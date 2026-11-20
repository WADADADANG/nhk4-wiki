# 02. Luau SDK Reference (`nhk.*` / `bot.*`)

All functions below are registered on the read-only global table `nhk` (aliased as `bot`).
Most functions support **Table Syntax** (`nhk.fn({ key = val })`) as the primary style and **Positional Syntax** (`nhk.fn(arg1, arg2)`).

---

## Module 1: Events & Triggers (`events.rs`)

### `nhk.onHotkey(opts | key, callback)`
Registers a callback triggered when a physical/DOM hotkey is pressed.
- **Params:** `key` (or `hotkey`, string e.g. `"F1"`, `"1"`, `"SPACE"`, `"CTRL+1"`), `callback` (or `fn`, function)
- **Example:**
```lua
nhk.onHotkey({ key = "F2", callback = function()
    nhk.log("info", "F2 pressed")
end })
```

### `nhk.onEvent(opts | eventName, callback)`
Listens for internal cross-profile events emitted by `nhk.emitEvent` or `/api/trigger/{event_name}`. Event names are normalized to uppercase.
- **Params:** `event` (or `name`, `eventName`, string), `callback` (or `fn`, `function(payload)`)

### `nhk.emitEvent(opts | eventName, [payload], [silent])` *(Alias: `nhk.emit`)*
Broadcasts a custom event with an optional JSON-serializable payload to all active profiles (Luau & Visual Node).
- **Params:** `event` (or `name`, `eventName`), `payload` (any Lua value/table, optional), `silent` (bool, default `false`)

### `nhk.onWebhook(opts | route, callback)`
Registers an inbound HTTP webhook listener at `GET/POST http://127.0.0.1:3088/api/webhook/{route}`. Route slugs are lowercased.
- **Params:** `route` (or `slug`, `path`, `name`), `callback` (or `fn`, `function(payload)`)

### `nhk.registerOverlayInput(opts)` *(Alias: `nhk.registerInput`)*
Registers a live UI control in the NHK4 Overlay panel and syncs its value to `variables_map`.
- **Params:** `name` (or `ref`, `id`), `text` (or `label`, `title`), `type` (`"number"`, `"boolean"`/`"bool"`, `"string"`/`"text"`), `default` (or `defaultValue`), `min`, `max`, `step`, `client` (or `slot`), `onChange` (or `callback`, optional `function(newValue)`)
- **Returns:** `boolean` (`true`)

### `nhk.onOverlayInput(opts | name, callback)` *(Alias: `nhk.onInput`)*
Subscribes to changes on a registered overlay input. Immediately invokes `callback(currentValue)` if a value already exists.
- **Params:** `name` (or `ref`, `id`), `callback` (or `fn`, `function(value)`)
- **Returns:** `boolean` (`true`)

---

## Module 2: Input & Mouse (`input.rs`)

### `nhk.pressKey(opts | client, key, [holdMs], [silent])`
Sends a background key press + release via CDP (or OS if global).
- **Params:** `client` (or `target`, `targetClient`: `1`, `"1,2"`, `"all"`), `key` (string), `hold` (or `holdMs`, `delay`, default `50`), `silent` (bool, default `false`)
- **Returns:** `boolean`

### `nhk.holdKey(opts | client, key, [mode], [silent])`
Holds, releases, or toggles a key on target client(s).
- **Params:** `client`, `key`, `mode` (or `action`: `"toggle"` default, `"hold"`/`"down"`/`"press"`/`"on"`/`"true"`, `"release"`/`"up"`/`"off"`/`"false"`), `silent` (bool)
- **Returns:** `boolean` (`true` if key is now held down, `false` if released)

### `nhk.forwardKey(opts | targets, key)`
Forwards a held physical key to target clients in real time while physically held.
- **Params:** `targets` (or `target`, `client`, default `"all"`), `key` (string), `isDown` (bool, optional), `trigger` (string, optional), `silent` (bool)
- **Returns:** `boolean`

### `nhk.mouseClick(opts | client, [button], [action], [x], [y], [holdMs], [silent])` *(Alias: `nhk.mouseMoveClick`)*
Dispatches a mouse click to a client (background CDP) or OS (`"global"` / `"0"` / `"os"`).
- **Params:**
  - `client` (or `target`, `targetClient`, default `"global"`)
  - `button` (or `btn`: `"left"` default, `"right"`, `"middle"`)
  - `action` (or `clickType`, `mode`: `"single"` default, `"double"`, `"down"`, `"up"`)
  - `x`, `y` (pixel or percentage string like `"50%"`, optional; if omitted on `"global"`, clicks at current cursor)
  - `rx`, `ry` (normalized ratio `0.0..1.0` or percentage, optional)
  - `relative` (or `clientRelative`, bool; defaults `true` for client slots, `false` for `"global"`)
  - `windowClient` (or `windowSlot`, `slot`: reference window for relative coords)
  - `onlyWhenFocused` (or `requireFocus`, bool, default `false`)
  - `hold` (or `holdMs`, default `25`)
  - `overlay` (or `showOverlay`, bool, default `false`)
  - `silent` (bool)
- **Returns:** `boolean`

### `nhk.autoClick(opts | ...)`
Starts/manages a background auto-clicker loop on a target client/coordinate.

### `nhk.mouseMove(opts | target, x, y)`
Moves the mouse cursor to `(x, y)` (supports `rx`/`ry`, `relative`, `windowClient`).
- **Returns:** `boolean`

### `nhk.mouseDrag(opts | client, button, fromX, fromY, toX, toY, [duration], [steps])`
Drags mouse smoothly from `(fromX, fromY)` to `(toX, toY)`.
- **Defaults:** `button = "left"`, `duration = 200` ms, `steps = 15`.
- **Returns:** `boolean`

### `nhk.focusClick(opts | client, [button], [x], [y])` *(Alias: `nhk.focusMoveClick`)*
Temporarily focuses the target OS window, performs an OS-level mouse click, and optionally restores previous focus & cursor position.
- **Params:** `client`, `button` (default `"left"`), `action` (`"single"`), `x`, `y`, `rx`, `ry`, `relative` (default `true`), `hold` (default `25`), `restoreFocus` (or `restore`, default `true`), `silent` (bool)
- **Returns:** `boolean`

### Client & Global Config Inspection (`client/config.rs`)
- **`nhk.getClientInfo(client)`** -> `{ slot, alias, online, paused, mode, browser, url, proxy, userAgent, appTarget?, bounds? } | nil`
- **`nhk.getClients()`** -> `table[]` (Array of `getClientInfo` tables for all configured `clientSlots`)
- **`nhk.getGlobalConfig()`** -> `{ activeProfile, activeProfiles, clientSlots, disabledClients, suspendHotkey, targetUrlKeyword, enableOverlay, enablePipOverlay, gpuAcceleration, webPort }`

---

## Module 3: Timing & Timers (`flow.rs`)

### `nhk.wait(opts | ms)` *(Alias: `nhk.sleep`)*
Async non-blocking sleep. Resets the VM instruction counter and aborts immediately with `ERR_ENGINE_STOPPED` if the engine stops or suspends.
- **Params:** `ms` (or `duration`, `delay`), `jitter` (optional random +/- ms)

### `nhk.setTimer(opts | callback, interval, [times], [name])`
Creates an MTA:SA-style non-blocking recurring or one-shot timer.
- **Params:** `callback` (or `fn`), `interval` (ms), `times` (or `repeat`, `count`: `0` = infinite, `1` = one-shot, `N` = N times), `name` (optional string ID), `jitter` (optional ms)
- **Returns:** `timerId` (string)

### `nhk.killTimer(timerId | opts)`
Stops and removes an active timer by ID/name.

### `nhk.isTimer(timerId | opts)`
Checks if a timer is currently active. Returns `boolean`.

### `nhk.resetTimer(timerId | opts)`
Resets a timer's elapsed interval.

---

## Module 4: Shared Variables (`state_vars.rs`)

Variables are shared across all active Luau & Visual Node profiles via `engine.variables_map`.

### `nhk.getVar(opts | name, [default])`
- **Params:** `name` (or `var`), `default` (or `defaultValue`, returned if not set)
- **Returns:** Value (number, boolean, string, table, or `nil`)

### `nhk.setVar(opts | name, value)`
- **Params:** `name` (or `var`), `value` (or `val`), `silent` (bool)
- **Returns:** Coerced value

### `nhk.toggleVar(opts | name)`
Flips a boolean variable (`false <-> true`).
- **Params:** `name` (or `var`), `silent` (bool)
- **Returns:** New `boolean` value

### `nhk.addVar(opts | name, [delta])`
Adds `delta` (default `1`) to a numeric variable.
- **Params:** `name` (or `var`), `delta` (or `amount`, `value`, default `1`), `silent` (bool)
- **Returns:** New `number` value

---

## Module 5: Client & Window Management (`client.rs`)

| Function | Aliases | Params | Returns |
| :--- | :--- | :--- | :--- |
| `nhk.launchClient` | `launch` | `client` / `slot` (number) | `boolean` |
| `nhk.closeClient` | `close`, `stop` | `client` / `slot` (number, `"1,2"`, or `"all"`) | `boolean` |
| `nhk.closeAllClients` | - | *(none)* | `boolean` |
| `nhk.isClientOnline` | - | `client` / `slot` (or `"global"`) | `boolean` |
| `nhk.saveFocus` | - | *(none)* | `hwnd` (`u64`) |
| `nhk.restoreFocus` | - | `hwnd` (`u64`) | `boolean` |
| `nhk.focusWindow` | - | `client` / `slot` / `hwnd` | `boolean` |
| `nhk.fakeFocus` | - | `client` / `slot` | `boolean` |
| `nhk.isWindowFocused` | - | `client` / `slot` | `boolean` |
| `nhk.getWindowInfo` | `getWindowState` | `client` / `slot` | Table `{ online, hwnd, isFocused, isMinimized, x, y, width, height, clientX, clientY, clientWidth, clientHeight }` |
| `nhk.setStatusOverlay` | `setStatus` | `{ client, name, status, detail, type, icon }` | `boolean` |
| `nhk.clearStatusOverlay` | `clearStatus` | `[{ client, names }]` (no args = clear all) | `boolean` |

---

## Module 6: Utilities, Audio, Webhooks & Safety (`utils.rs`)

### `nhk.log(...)`
Writes a message to the NHK4 log stream. (Rule: use ASCII English for system logs).
- **Table Syntax:** `nhk.log({ message = "...", level = "info", silent = false })`
- **Positional Syntax:** `nhk.log("info", "Hello")` or `nhk.log("Hello")`
- **Levels:** `"action"`, `"info"`, `"system"`, `"warn"`, `"error"`, `"log"`, `"success"`

### `nhk.playSound(opts | sound, [volume])`
Plays a sound file from `public/sounds/` (defaults to `.wav` if no extension).
- **Params:** `sound` (or `file`, `name`), `volume` (or `vol`, `0..100` or `0.0..1.0`, default `100`)

### `nhk.speakTts(opts | text, [voice])` *(Pro)*
Synthesizes and plays Edge TTS audio asynchronously.
- **Params:** `text` (or `msg`), `voice` (default `"th-TH-PremwadeeNeural"`), `silent` (bool)
- **Returns:** `boolean`

### `nhk.sendWebhook(opts | url, [message], [image])`
Sends an async HTTP POST webhook (JSON, plain text, or multipart image upload).
- **Params:** `url`, `message` (or `content`), `image` (or `imagePath`, `file`)
- **Returns:** `boolean`

### `nhk.emergencyStop([opts | reason])`
Immediately halts the engine (or a specific scope).
- **Params:** `reason` (or `message`), `scope` (`"all"` default), `client`, `node_id` (or `nodeId`), `profile` (or `profile_name`)

### `nhk.httpRequest(opts | url, [method], [body])` *(Feature: `http_client`)*
Sends an HTTP request (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`) and auto-parses JSON responses.
- **Params:** `url`, `method` (`"GET"` default), `headers` (table), `body` (string or table — auto-serialized as JSON), `timeoutMs` (`10000` default)
- **Returns:** `{ status, ok, body, json?, error? }`

### `nhk.getClipboard()` / `nhk.setClipboard(opts | text)`
Reads or writes plain UTF-8 text on the Windows Clipboard.
- `getClipboard()` -> `string`
- `setClipboard({ text })` -> `boolean`

### Local File & JSON Storage (`storage.rs`)
Supports **100% user-specified paths** (e.g. `"D:/Data/out.json"`, `"C:/Users/%USERNAME%/Documents/NHK4/data.json"`, or `"~/Documents/NHK4/data.json"`). Relative paths resolve from the current working directory. Parent directories are created automatically.
- `nhk.writeFile({ path, content, append? })` -> `boolean`
- `nhk.readFile({ path })` -> `string | nil`
- `nhk.writeJson({ path, data, pretty? })` -> `boolean`
- `nhk.readJson({ path })` -> `table | any | nil`

### Custom HTML/CSS/JS Overlays (`custom_overlay.rs`)
Creates independent floating overlay windows with 2-way Luau ↔ HTML/JS messaging. Multiple overlays can run simultaneously with unique `id`s and are automatically closed when the script stops.
- `nhk.createOverlay(id, htmlOrOpts)` -> `boolean`
  `htmlOrOpts` can be a raw HTML string or `{ html?, url?, title?, width?, height?, x?, y?, transparent?, alwaysOnTop?, clickThrough? }`.
- `nhk.sendOverlay(id, event, data?)` -> `boolean`
  Pushes `{ id, event, data }` to the overlay page (handled in JS via `window.nhkOverlay.onMessage(fn)` or `window.addEventListener('message', ...)`).
- `nhk.closeOverlay(id)` -> `boolean`
- `nhk.onOverlayMessage(id, callback)` -> `boolean`
  Registers a Luau callback `function(msg)` receiving `{ id, event, data }` sent from JS via `window.nhkOverlay.send(event, data)`.

---

## Modules 7, 8, 9: Vision, Monster, Target & Vision Primitives (`vision/`)
See **[04. Vision, Party & Monster Target](04-Vision-Party-And-Monster-Target.md)** for full parameter and return field documentation (including `captureFrame`, `getPixel`, `findColor`, `countColor`, `findRowBar`, `compareRegion`, `findImage`, `saveFrame`, and `ocrRegion`).

---

## Module 10: Web DOM, WebSocket & WASM (`web/`) *(Feature: `web_dom`)*

> **When to use Web DOM vs Vision:**
> - `webEval`, `webQuery`, `webWaitFor`, `webClick`, `webType` operate on **standard HTML DOM elements** (normal websites, dashboards, forms, item tables).
> - For **WebGL / `<canvas>` games**, DOM selectors cannot see inside the canvas — use **Vision Primitives** (`captureFrame`, `findRowBar`, `ocrRegion`) or **Opt-In WASM** (`wasmEnable`, `wasmScan`, `wasmRead`).

### Web DOM Extraction & Control (`web/dom.rs`)
- **`nhk.webEval({ client, js })` -> `any`**
  Executes JavaScript inside the client browser tab via CDP `Runtime.evaluate` (supports `Promise` awaiting) and converts JSON return values into Luau tables/primitives.
- **`nhk.webQuery({ client, selector, attr?, all? })` -> `string | string[] | nil`**
  Queries `document.querySelector` (or `querySelectorAll` if `all = true`). Omit `attr` for `innerText`, or pass `"value"`, `"href"`, `"src"`, `"innerHTML"`, etc.
- **`nhk.webWaitFor({ client, selector, timeoutMs? })` -> `boolean`**
  Event-driven wait using `MutationObserver` inside the page until `selector` exists or `timeoutMs` (default `5000`) expires.
- **`nhk.webClick({ client, selector })` -> `boolean`**
  Scrolls the first matching element into view, focuses it, and calls `.click()`.
- **`nhk.webType({ client, selector, text, clear? })` -> `boolean`**
  Sets input/textarea value using native property setters and dispatches `input` & `change` events (compatible with React/Vue controlled inputs). `clear` defaults to `true`.
- **`nhk.webInfo({ client })` -> `{ url, title } | nil`**
  Returns the current tab URL and `document.title`.
- **`nhk.webNavigate({ client, url })` -> `boolean`**
  Navigates the client browser tab to `url` (`http://` or `https://`) via CDP `Page.navigate`.

### Web Canvas Draw Overlay (`web/draw.rs`)
Click-through `<canvas>` overlay (`pointer-events: none`) inside the client browser viewport using exact viewport coordinates (`durationMs` defaults to `1500`, pass `0` for persistent):
- `nhk.drawRect({ client, x, y, w, h, color?, thickness?, fill?, label?, durationMs?, id? })` -> `boolean`
- `nhk.drawLine({ client, x1, y1, x2, y2, color?, thickness?, durationMs?, id? })` -> `boolean`
- `nhk.drawText({ client, x, y, text, color?, size?, bg?, durationMs?, id? })` -> `boolean`
- `nhk.clearDraw(client, id?)` -> `boolean`

### Opt-In CDP WebSocket / Network Capture (`web/network.rs`)
Zero footprint when not called; activates `Network.enable` only on the requested client slot.
- **`nhk.netStart({ client })` -> `boolean`** — Enables CDP Network events and starts buffering WebSocket frames (up to 512 events).
- **`nhk.netPoll({ client, limit? })` -> `table[]`** — Drains queued events `{ type = "ws_open"|"ws_recv"|"ws_send", requestId, url?, opcode, payload, timestampMs }`.
- **`nhk.netStop({ client })` -> `boolean`** — Calls `Network.disable` and clears the buffer.

### Opt-In WASM Memory Inspector (`web/wasm.rs`)
Zero injection at browser startup; only instruments `WebAssembly` (using a non-enumerable `Symbol` store across main window and same-origin iframes) when `wasmEnable` is called.
- **`nhk.wasmEnable({ client, reload? })` -> `boolean`** — Arms the tracker; pass `reload = true` if the WASM module already loaded before enabling.
- **`nhk.wasmDisable({ client })` -> `boolean`** — Removes `addScriptToEvaluateOnNewDocument` and deletes the tracker reference.
- **`nhk.wasmInfo({ client })` -> `{ enabled, count, memories: [{ index, bytes, pages }] }`** — Lists captured `WebAssembly.Memory` buffers (1 page = 65,536 bytes).
- **`nhk.wasmScan({ client, value?, type?, compare?, tolerance?, index?, offsets?, maxResults? })` -> `number[]`** — Scans for `value` (`type`: `"i32"`, `"u32"`, `"i16"`, `"f32"`, `"f64"`, `"string"`, `tolerance` default `0.5`). Supports `compare`: `"exact"` *(default)*, `"unknown"` *(initial snapshot)*, `"changed"`, `"unchanged"`, `"increased"`, `"decreased"`. Pass `offsets = previousHits` to narrow down (**Next Scan**).
- **`nhk.wasmRead({ client, offset, type?, length?, index? })` -> `number | string | number[] | nil`** — Reads typed value (`"i32"`, `"u32"`, `"i16"`, `"u8"`, `"f32"`, `"f64"`, `"string"`, `"bytes"`) at byte `offset`.

---

## Module 11: Windows Desktop App UI Automation (`app/`) *(Feature: `app_uia`)*
Allows scripts to inspect and automate **native Windows desktop applications** (non-browser apps like Calculator, Notepad, launchers, etc.) via Windows `IUIAutomation`.
- **`nhk.appListWindows({ filter? })` -> `table[]`**
  Lists visible top-level windows with `{ hwnd, pid, title, processName, className, rect = { x, y, w, h } }`. Optional `filter` matches title/process/class case-insensitively.
- **`nhk.appReadText({ hwnd?, title?, process?, controlType?, name?, maxDepth?, maxResults? })` -> `table[] | nil`**
  Traverses the UI Automation tree (default `maxDepth = 6`, `maxResults = 100`) and returns controls with text/values: `{ name, value, controlType, automationId, rect }`.
- **`nhk.appClickControl({ hwnd?, title?, process?, name?, automationId?, controlType?, maxDepth? })` -> `boolean`**
  Finds a control by `name` or `automationId` and invokes `InvokePattern` (or clicks its center rect as fallback).


