# NHK4 External Developer & AI Knowledge Wiki

Welcome to the official **NHK4 (NodeHotKeyV4)** standalone documentation Wiki.

This Wiki is designed to be uploaded to **GitHub / GitHub Wiki** so users and developers can feed it directly to **external AI assistants** (ChatGPT, Claude, Gemini, Cursor, etc.) to teach them:
1. How NHK4 works internally (Rust Native Core, Sandboxed Luau VM, CDP Browser Automation, Event-Driven Architecture).
2. Exact signatures, parameters, and return schemas of the **`nhk.*` / `bot.*` Luau SDK**.
3. How to connect external apps, Stream Decks, Python/Node.js scripts, or webhooks to NHK4 via **HTTP REST & SSE (`127.0.0.1:3088`)**.
4. How to use **Computer Vision** (Party HP Scanner, Monster Scanner, Target Status, Screencast RAM Stream) and **Overlay Inputs**.

---

## Quick Navigation

| Page | Description |
| :--- | :--- |
| [01. Architecture & Quickstart](01-Architecture-And-Quickstart.md) | Core architecture, Dynamic Client Slots, Luau Sandbox rules, Event-Driven execution |
| [02. Luau SDK Reference (`nhk.*`)](02-Luau-SDK-Reference.md) | Complete reference for all 9 SDK modules (`events`, `input`, `flow`, `state_vars`, `client`, `utils`, `vision`, `vision_monster`, `vision_target`) |
| [03. REST API, Webhooks & Events](03-REST-API-Webhooks-And-Events.md) | External HTTP endpoints (`127.0.0.1:3088`), `/api/trigger/{event}`, `/api/webhook/{route}`, SSE streams |
| [04. Vision, Party & Monster Target](04-Vision-Party-And-Monster-Target.md) | `scanParty`, `partyHeal`, `partyBuffNext`, `selectPartySlot`, `scanMonsters` (Real OCR + Red Aggro PNG Clip), `findMonsterBody`, `isMonsterHovered`, `getTarget`, `scanCharStatus`, Minimap Drift, RAM Snaps, Vision Primitives & OCR |
| [05. Overlay Input & Multi-Client](05-Overlay-Input-And-MultiClient.md) | Dynamic Overlay UI controls (`registerOverlayInput`, `onOverlayInput`), window focus, `fakeFocus`, status overlays, Canvas Drawing (`drawRect`, `drawCircle`, `drawLine`, `drawText`, `clearDraw`) |
| [06. Examples & AI Prompt Templates](06-Examples-And-AI-Prompt-Templates.md) | Ready-to-use Luau script recipes and copy-paste prompts for external AIs |
| [07. Visual Node Mode & Node Reference](07-Visual-Node-Mode-And-Connections.md) | Blueprint node graph JSON schema, Exec/Data connections, and complete reference of all visual nodes |
| [AI Context Bundle (Single File)](AI_CONTEXT_BUNDLE.md) | Compact all-in-one reference file you can upload or paste into any AI chat in one shot |

---

## How to Use This Wiki with an External AI

1. **Single-file fast method:** Upload or paste [`AI_CONTEXT_BUNDLE.md`](AI_CONTEXT_BUNDLE.md) into ChatGPT / Claude / Gemini, then describe the script or integration you want.
2. **Deep-dive method:** Attach [`02-Luau-SDK-Reference.md`](02-Luau-SDK-Reference.md) + the relevant topic page (e.g., [`04-Vision-Party-And-Monster-Target.md`](04-Vision-Party-And-Monster-Target.md) or [`03-REST-API-Webhooks-And-Events.md`](03-REST-API-Webhooks-And-Events.md)).
3. **Copy-paste starter prompt:** See [`06-Examples-And-AI-Prompt-Templates.md`](06-Examples-And-AI-Prompt-Templates.md).

