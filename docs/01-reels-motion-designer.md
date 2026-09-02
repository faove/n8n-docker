# Etapa 1 — `reels-motion-designer`

Origen: `.claude/agents/reels-motion-designer.md` (proyecto OtroGalloMarketing).
Workflow: [`workflows/agentes-content-engine/reels-motion-designer.json`](../workflows/agentes-content-engine/reels-motion-designer.json).

## Qué hace este workflow

Recibe un pedido en lenguaje natural + contexto de marca por webhook, llama a la API
de Claude forzando una salida estructurada (tool use), y devuelve en la misma
respuesta HTTP:

- El guion + shot list en Markdown (hook, escenas, ritmo, CTA, música/SFX, fuentes).
- El storyboard animado: un HTML autocontenido (9:16, CSS embebido, `@keyframes`) por
  cada escena del guion.
- Qué datos de marca quedaron marcados como `pendientes` en vez de inventarse.

No genera vídeo renderizado ni publica nada — igual que el subagente original, esto es
insumo para revisión humana.

## Qué cambió respecto al subagente original (y por qué)

- **Contexto de marca**: el subagente original lee `_Contexto/estrategia.md`,
  `_BrandKit/` y `_Clientes/<slug>/` del filesystem. El workflow no tiene filesystem
  compartido (es multi-cliente vía API), así que ese mismo contexto viaja **inline en
  el body del webhook** (`marca`, `estrategia`, `cliente`). Si un campo no viene en el
  request, el nodo `Normalizar Input` lo detecta y lo pasa como pendiente — el system
  prompt le prohíbe a Claude inventarlo.
- **`DesignSync`**: no existe fuera de Claude Code. V1 no sincroniza nada a Claude
  Design — los HTML de las escenas se devuelven directamente en la respuesta JSON del
  webhook. Si más adelante hace falta una superficie de revisión compartible, se agrega
  un paso de subida a storage externo (S3/Drive) sin tocar el resto del workflow.
- **`Write` (guardar en disco)**: no aplica en un SaaS multi-cliente sin filesystem
  compartido. El resultado completo (guion + escenas) se devuelve en la respuesta del
  webhook; persistencia real (DB/storage por cliente) queda para una fase posterior.
- **Salida estructurada**: en vez de pedirle a Claude que devuelva Markdown + N bloques
  HTML mezclados en texto libre (frágil de parsear), el nodo `Claude API - Reels
  Storyboard` fuerza `tool_choice: {"type":"tool","name":"deliver_reel_storyboard"}`
  contra un `input_schema` que ya define `guion_markdown`, `escenas[]`, `pendientes[]`
  y `fuentes[]`. El nodo `Extraer Resultado` sólo lee ese bloque `tool_use` — no hay
  parsing de Markdown con regex.

## Contrato de entrada (`POST /webhook/reels-motion-designer`)

```json
{
  "brief": "Créame un reel para mi autolavado con 20% de descuento en septiembre",
  "cliente": { "nombre": "Autolavado Brillo", "slug": "autolavado-brillo" },
  "marca": {
    "tono": "cercano, directo, con humor moderado",
    "visual": {
      "paleta": ["#0B5FFF", "#FFFFFF", "#111111"],
      "tipografia": "Poppins",
      "estilo": "fotográfico, alto contraste"
    },
    "claims_aprobados": ["Lavado en 15 minutos", "Sin turno previo"]
  },
  "estrategia": {
    "pilares_contenido": ["promociones", "prueba social"],
    "oferta": "20% off en lavado completo durante septiembre",
    "etapa_funnel": "conversión"
  }
}
```

Único campo obligatorio: `brief`. Todo lo demás es opcional — lo que falte se marca
como pendiente en la respuesta en vez de inventarse.

## Contrato de salida

```json
{
  "ok": true,
  "cliente": { "nombre": "Autolavado Brillo", "slug": "autolavado-brillo" },
  "brief": "...",
  "guion_markdown": "# Reel — ...",
  "escenas": [
    { "numero": 1, "titulo": "Hook", "html": "<!doctype html>..." }
  ],
  "pendientes": [],
  "fuentes": ["marca.tono", "estrategia.oferta"]
}
```

Si la llamada a Claude falla o no devuelve el `tool_use` esperado, la respuesta es
`{ "ok": false, "error": "..." }` (con código 200 igual — el contrato de error es a
nivel de payload, no de status HTTP, para que el front pueda distinguir siempre un
JSON parseable).

## Diseño de nodos

1. **Webhook** (`POST /reels-motion-designer`, `responseMode: responseNode`).
2. **Normalizar Input** (Code): valida `brief`, aplica defaults a `marca`/`estrategia`/
   `cliente`, calcula `campos_faltantes_input`.
3. **Construir Prompt** (Code): arma el mensaje de usuario (brief + resumen de
   contexto) y el body completo para la API de Claude, incluyendo el `system` prompt,
   el `tool` schema y `tool_choice` forzado.
4. **Claude API - Reels Storyboard** (HTTP Request): `POST
   https://api.anthropic.com/v1/messages`, credencial `httpHeaderAuth` (`x-api-key`),
   header `anthropic-version: 2023-06-01`, modelo `claude-sonnet-4-5`, `max_tokens: 8000`.
5. **Extraer Resultado** (Code): busca el bloque `tool_use` con `name ===
   "deliver_reel_storyboard"` y arma la respuesta final; si no aparece, devuelve
   `ok:false` con el `stop_reason`.
6. **Responder Webhook** (`respondToWebhook`): devuelve el JSON tal cual.

## Pendientes / pasos manuales antes de poder probarlo

- [x] Crear en `n8n.devnodo.com` una credencial **HTTP Header Auth** con header
      `x-api-key: <API key real>` (nombre final en n8n: `Header Auth account`,
      id `ivahrU1eTJtVFp4J`).
- [x] Re-asignar esa credencial en el nodo `Claude API - Reels Storyboard`.
- [x] Importar `workflows/agentes-content-engine/reels-motion-designer.json` en
      `n8n.devnodo.com` (workflow id `9OLyb3gqijYNLgGT`) y probar con el payload de
      ejemplo.
- [ ] Validar que el HTML de cada escena renderiza razonablemente en 9:16 (abrir uno
      de los HTML devueltos en el navegador) — probado solo por payload/JSON, falta
      revisión visual.
- [ ] Decidir si el `max_tokens: 8000` alcanza para guiones con muchas escenas o si
      conviene ajustar por longitud del brief.

## Bugs encontrados y corregidos durante la prueba (2026-09-02)

- **JSON sin `id` de nivel superior**: el `n8n import:workflow` de esta versión
  (2.36.9) lo exige; sin él fallaba con `null value in column "id"`. Se agregó un id
  generado (`9OLyb3gqijYNLgGT`).
- **Nodo `Claude API - Reels Storyboard` sin manejo de errores**: cualquier respuesta
  no-2xx de Anthropic (ej. 401) rompía toda la ejecución y el webhook devolvía vacío,
  en vez del `{ok:false, error:...}` que ya sabe armar `Extraer Resultado`. Se agregó
  `options.response.response.neverError: true` para que el nodo pase la respuesta de
  error en vez de tirar la ejecución.
- **Credencial mal configurada por error de UI**: se cargó el nombre del header como
  el título de la credencial (`devnodo-key`) en vez de en el campo interno `Name`
  (que es el header HTTP real). Corregido a `Name: x-api-key`.
- **Password del rol Postgres `n8n` desincronizada** (bug de infra, no del workflow):
  `DB_POSTGRESDB_PASSWORD` (usada por el contenedor n8n) y `POSTGRES_NON_ROOT_PASSWORD`
  (usada por `init-data.sh` al crear el rol la primera vez) no coincidían en el `.env`
  de `/srv/projects/n8n-devnodo` en el servidor — causaba `password authentication
  failed for user "n8n"` intermitente en cualquier ejecución de n8n, no solo en este
  workflow. Corregido con `ALTER ROLE` para sincronizar el password real del rol con
  el de `.env`. Vale la pena revisar ese `.env` si en el futuro se edita alguna de las
  dos variables sin correr el `ALTER ROLE` correspondiente (postgres solo aplica la
  contraseña del rol al crear el volumen la primera vez).

## Estado

✅ **Importado y probado en `n8n.devnodo.com`** — workflow activo (id
`9OLyb3gqijYNLgGT`), credencial real conectada, probado end-to-end contra la API real
de Claude con el payload de ejemplo, respuesta `ok:true` estable en corridas
sucesivas. Queda pendiente la revisión visual del HTML de las escenas en navegador y
la decisión sobre `max_tokens`.
