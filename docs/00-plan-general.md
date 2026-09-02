claude# Plan — Automatizaciones de subagentes en n8n (base de ai.devnodo.com)

## Objetivo

Convertir cada subagente definido en `.claude/agents/` del proyecto OtroGalloMarketing
(`/home/faove/DevNodo/OtroGalloMarketing/.claude/agents/`) en un workflow de n8n
autónomo, desplegado en `n8n.devnodo.com`, que llama a la API de Claude directamente
(sin depender de Claude Code ni de un filesystem compartido).

Estos workflows son la base técnica de un producto **aparte y general**:
`ai.devnodo.com` — un SaaS multi-cliente donde el usuario escribe un pedido en
lenguaje natural (ej. *"Créame un reel para mi autolavado con 20% de descuento en
septiembre"*) y el sistema entrega estrategia, guion, video, copy y hashtags listos
para revisar. Por eso cada workflow se diseña **genérico desde el día uno**: nada de
rutas de OtroGallo hardcodeadas, nada de lectura de archivos locales — el contexto de
marca/cliente viaja siempre en el payload de entrada.

## Diferencias clave: subagente Claude Code → workflow n8n

| En `.claude/agents/*.md` (Claude Code) | En n8n |
|---|---|
| Lee `_Contexto/estrategia.md`, `_BrandKit/`, `_Clientes/<slug>/` del filesystem | Recibe el mismo contexto **inline en el JSON** del webhook (brand kit, brief, tono) — no hay filesystem compartido en el SaaS |
| Tools nativas de Claude Code (`Write`, `DesignSync`, `Read`, `Glob`) | Nodo **HTTP Request** directo contra `https://api.anthropic.com/v1/messages`, con **tool use forzado** (`tool_choice`) para obtener salida estructurada (JSON) en vez de Markdown suelto |
| `DesignSync` sincroniza el storyboard HTML con un proyecto de Claude Design | **No existe fuera de Claude Code.** V1: el workflow devuelve el/los HTML directamente en la respuesta del webhook (y opcionalmente los persiste). Revisión visual se resuelve fuera de Claude Design por ahora — ver decisión abajo |
| Guarda archivos en rutas fijas del repo | Devuelve el resultado en la respuesta HTTP (JSON) del webhook; persistencia a storage real (S3/Drive/DB) queda para una fase posterior, no v1 |

## Decisiones de arquitectura (confirmadas con el usuario, 2026-09-01)

1. **Trigger/input**: Webhook JSON. Cada workflow expone un `POST` que recibe el
   brief en lenguaje natural + brand kit/contexto de cliente inline. Es el contrato
   que después usará el front de `ai.devnodo.com`.
2. **Storyboard/DesignSync**: sin integración con Claude Design en v1. El workflow de
   `reels-motion-designer` devuelve el guion + el HTML de cada escena directamente en
   el JSON de respuesta. Si más adelante se necesita una superficie de revisión
   compartible, se evalúa subir esos HTML a storage externo (S3/Drive) como paso
   adicional — no bloquea v1.
3. **Llamada a Claude**: nodo **HTTP Request** directo a la Messages API de Anthropic
   (no el nodo nativo LangChain de n8n), usando `tools` + `tool_choice` forzado para
   garantizar una salida estructurada (guion + N escenas en un solo JSON), en vez de
   parsear Markdown libre.

Estas tres decisiones aplican como default a **todos** los agentes de esta lista salvo
que la ficha de una etapa puntual documente una excepción.

## Convenciones comunes a todos los workflows

- **Credencial Claude**: una sola credencial HTTP Header Auth (`x-api-key`) en n8n,
  reutilizada por todos los workflows de este plan. Nombre sugerido en n8n:
  `Anthropic API Key`.
- **Modelo**: `claude-sonnet-4-5` salvo que la tarea pida explícitamente otra cosa.
- **Idioma de salida**: español (igual que el subagente original), salvo el caso ya
  documentado de `youtube-thumbnail-prompter` (el prompt de imagen va en inglés por
  requisito técnico de las herramientas de generación de imagen).
- **Contrato de error**: si falta contexto de marca indispensable (paleta, tono,
  claims aprobados), el workflow no debe inventarlo — debe devolverlo marcado como
  `pendiente` en la respuesta, igual que hace el subagente original vía system prompt.
- **Nomenclatura de archivos de workflow**: `workflows/agentes-content-engine/<slug-agente>.json`.
- **Nomenclatura de docs de etapa**: `docs/<NN>-<slug-agente>.md`.

## Etapas (una por subagente, se ejecutan una a la vez)

| # | Subagente origen | Workflow n8n | Estado |
|---|---|---|---|
| 1 | `reels-motion-designer.md` | `workflows/agentes-content-engine/reels-motion-designer.json` | ✅ importado y probado en `n8n.devnodo.com` — ver [01-reels-motion-designer.md](01-reels-motion-designer.md) |
| 2 | `youtube-script-write.md` | `workflows/agentes-content-engine/youtube-script-write.json` | ⚪ pendiente |
| 3 | `youtube-title-generator.md` | `workflows/agentes-content-engine/youtube-title-generator.json` | ⚪ pendiente |
| 4 | `youtube-thumbnail-prompter.md` | `workflows/agentes-content-engine/youtube-thumbnail-prompter.json` | ⚪ pendiente |
| 5 | Orquestador (fase futura) | Un webhook único que recibe el pedido en lenguaje natural del usuario final y decide a qué workflow(s) de arriba llamar (equivalente al futuro flujo de `ai.devnodo.com`) | ⚪ pendiente — no arrancar hasta tener los 4 agentes base funcionando sueltos |

Leyenda: ⚪ pendiente · 🟡 en curso · 🟢 hecho en n8n local · ✅ importado y probado en `n8n.devnodo.com`

## Cómo seguir el avance

Cada etapa tiene su propio archivo `docs/<NN>-<slug>.md` con: contrato de entrada/salida,
diseño de nodos, decisiones específicas de ese agente, y checklist de estado (diseñado →
importado a n8n local → probado con la API real → subido a `n8n.devnodo.com`). Actualizar
ese checklist a medida que se avanza es lo que permite retomar el trabajo sin releer todo
el hilo.
