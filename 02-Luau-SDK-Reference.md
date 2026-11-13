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

---

## Modules 7, 8, 9: Vision, Monster & Target (`vision.rs`, `vision_monster.rs`, `vision_target.rs`)
See **[04. Vision, Party & Monster Target](04-Vision-Party-And-Monster-Target.md)** for full parameter and return field documentation.

