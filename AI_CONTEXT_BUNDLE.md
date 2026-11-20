# NHK4 (NodeHotKeyV4) — Master AI Context Bundle

> **Purpose:** Feed this single file to any external AI (ChatGPT, Claude, Gemini, Cursor) so it knows the exact NHK4 architecture, Luau SDK (`nhk.*`), Vision APIs, and HTTP REST/Webhook endpoints without hallucinating APIs.

---

## 1. Core Architecture & Rules
- **Runtime:** Native Rust + Tokio engine + Axum HTTP/SSE server on **`http://127.0.0.1:3088`** + Sandboxed Luau VM (`mlua`) + CDP Browser Control + OS Input (`rdev`/Win32).
- **Fixed Client Slots:** Game windows are bound to permanent numeric slot IDs (`1`, `2`, `3`, ...). Targets can be `1`, `"1,2"`, `"all"`, or `"global"` / `"0"` / `"os"` (host OS).
- **Sandbox:** `nhk` (aliased as `bot`) is a read-only global SDK table. `io` and OS shell commands are blocked.
- **No Busy Loops:** The VM enforces `200,000` instructions max without yielding (`ERR_INSTRUCTION_LIMIT`). Always use **event callbacks** (`onHotkey`, `onEvent`, `onWebhook`, `onOverlayInput`) and **non-blocking timers** (`setTimer`), or yield with `nhk.wait({ ms = ... })`.
- **Dual Syntax:** Most functions accept both Table Syntax (`nhk.pressKey({ client = 1, key = "F1" })`) and Positional Syntax (`nhk.pressKey(1, "F1")`). Add `silent = true` in table args inside fast timers to avoid log spam.
- **Pro License:** Vision functions (`scanParty`, `partyHeal`, `partyBuffNext`, `selectPartySlot`, `takeScreenshot`, `takeDesktopScreenshot`, `scanMonsters`, `targetMonster`, `scanMonstersStream`, `getTarget`, `getTargetStream`) and `speakTts` require a Pro license.

---

## 2. Complete Luau SDK (`nhk.*` / `bot.*`)

### Events & Overlay Inputs
- `nhk.onHotkey({ key, callback })` — Register hotkey trigger (`"F1"`, `"1"`, `"CTRL+1"`, etc.).
- `nhk.onEvent({ event, callback })` — Listen for internal cross-profile events (uppercase normalized). `callback(payload)`.
- `nhk.emitEvent({ event, payload, silent })` *(alias `emit`)* — Broadcast event to all active Luau & Visual Node profiles.
- `nhk.onWebhook({ route, callback })` — Listen for HTTP `GET/POST /api/webhook/{route}`. `callback(payload)`.
- `nhk.registerOverlayInput({ name, text, type, default, min, max, step, client, onChange })` *(alias `registerInput`)* — `type`: `"number" | "boolean" | "string"`. Syncs with `variables_map`.
- `nhk.onOverlayInput({ name, callback })` *(alias `onInput`)* — Fires immediately with current value + on change.

### Input & Mouse
- `nhk.pressKey({ client, key, hold, silent })` — Background key tap (`hold` default `50` ms). Returns `bool`.
- `nhk.holdKey({ client, key, mode, silent })` — `mode`: `"toggle"` (default), `"hold"`/`"down"`, `"release"`/`"up"`. Returns `bool` (`isDown`).
- `nhk.forwardKey({ targets, key, isDown, trigger, silent })` — Real-time key forwarder while held.
- `nhk.mouseClick({ client, button, action, x, y, rx, ry, relative, windowClient, onlyWhenFocused, hold, overlay, silent })` *(alias `mouseMoveClick`)* — `button`: `"left"|"right"|"middle"`, `action`: `"single"|"double"|"down"|"up"`. `x,y` support px or `"50%"`.
- `nhk.autoClick(...)` — Background auto-clicker.
- `nhk.mouseMove({ target, x, y, rx, ry, relative, windowClient })`
- `nhk.mouseDrag({ client, button, fromX, fromY, toX, toY, duration, steps, relative })` — Defaults `duration=200`, `steps=15`.
- `nhk.focusClick({ client, button, action, x, y, rx, ry, relative, hold, restoreFocus, silent })` *(alias `focusMoveClick`)* — Temporarily focuses window for OS click, then restores focus & cursor.

### Timing & Timers
- `nhk.wait({ ms, jitter })` *(alias `sleep`)* — Async yield; aborts on Engine Stop (`ERR_ENGINE_STOPPED`).
- `nhk.setTimer({ name, interval, times, jitter, callback })` — `times=0` infinite, `1` one-shot. Returns `timerId`.
- `nhk.killTimer(timerId)` / `nhk.isTimer(timerId)` / `nhk.resetTimer(timerId)`

### Shared Variables (`engine.variables_map`)
- `nhk.getVar({ name, default })`
- `nhk.setVar({ name, value, silent })`
- `nhk.toggleVar({ name, silent })` -> `bool`
- `nhk.addVar({ name, delta, silent })` -> `number`

### Client & Window Management
- `nhk.launchClient(slot)` *(alias `launch`)* -> `bool`
- `nhk.closeClient(slot | "1,2" | "all")` *(aliases `close`, `stop`)* -> `bool`
- `nhk.closeAllClients()` -> `bool`
- `nhk.isClientOnline(slot | "global")` -> `bool`
- `nhk.saveFocus()` -> `hwnd (u64)` | `nhk.restoreFocus(hwnd)` -> `bool`
- `nhk.focusWindow(slot | hwnd)` -> `bool` | `nhk.fakeFocus(slot)` -> `bool` | `nhk.isWindowFocused(slot)` -> `bool`
- `nhk.getWindowInfo(slot)` *(alias `getWindowState`)* -> `{ online, hwnd, isFocused, isMinimized, x, y, width, height, clientX, clientY, clientWidth, clientHeight }`
- `nhk.setStatusOverlay({ client, name, status, detail, type, icon })` *(alias `setStatus`)*
- `nhk.clearStatusOverlay({ client, names })` *(alias `clearStatus`; no args = clear all)*

### Vision, Party, Monster & Target (Pro)
- `nhk.startStream({ client, quality })` / `nhk.stopStream({ client })` — Starts/stops RAM screencast (`quality` `20..100`, default `65`) for `<10ms` `*Stream` vision calls.
- `nhk.scanParty({ client, region, threshold, overlay, silent })` -> `{ count, aliveCount, totalSlots, lowestHp, hasLowHp, lowestSlot, members, details }` (`region` default `"right"`, `threshold` default `70.0`).
- `nhk.partyHeal({ client, threshold, key, region, clickTarget, showOverlay, delayAfterClick, silent })` -> `bool`
- `nhk.partyBuffNext({ client, key, region, silent })` -> `bool`
- `nhk.selectPartySlot({ client, slot, region, overlay, delay, silent })` -> `number` (`slot 1..8` or `0`)
- `nhk.scanMonsters({ client, name, priority, offsetY, maxDistance, minScore, skipCenter, skipPlayers, overlay, quality, silent })` -> `{ count, aggroCount, passiveCount, hasTarget, targetHp, isMonsterTarget, best, monsters }` *(Real Windows OCR name verification cached 15s per `trackId`, with automatic lossless PNG clip for uncached Red Aggro nameplates)*
- `nhk.scanMonstersStream(...)` — Same as `scanMonsters` using RAM stream (`<10ms`, includes `latencyMs`).
- `nhk.targetMonster({ client, name, attackKey, priority, offsetY, maxDistance, minScore, skipCenter, skipPlayers, click, delayAfterClick, overlay, silent })` -> `{ found, success, count, aggroCount, passiveCount, hasTarget, targetHp, name, x, y, clickX, clickY, isAggro, distance, dist, score, trackId, id }`
- `nhk.findMonsterBody({ client, mob | x, y, width, height, padX?, offsetY?, scanHeight?, motionWeight?, useMotion?, maskPlayer?, maxCandidates?, overlay?, overlayDurationMs?, quality?, silent? })` -> `{ found, clickX, clickY, areaPixels, coreDepth, confidence, latencyMs, bodyBox, roiBox, candidates }`
- `nhk.findMonsterBodies(client, mobsList, opts?)` -> `bodyResult[]` *(Batch body detection from 1 frame capture)*
- `nhk.isMonsterHovered({ client, mob | x, y, width, height, padX?, offsetY?, scanHeight?, minPixels?, fresh?, maxFrameAgeMs?, quality?, silent? })` -> `{ hovered, outlinePixels, latencyMs, box }`
- `nhk.getTarget({ client, log, overlay, quality })` / `nhk.getTargetStream({ client, log, overlay })` -> `{ hasTarget, className, hp, isDead, isMonster, hasNumbers, hasMpBar, circleType, scale, hasGoldX, x, y, width, latencyMs }`
- `nhk.scanCharStatus({ client, position?, scale?, minScale?, maxScale?, region?, ocr?, ocrLang?, autoHideNumbers?, useStream?, overlay?, guide?, overlayDurationMs?, quality?, silent? })` / `nhk.scanCharStatusStream(...)` -> `{ found, zone, scale, confidence, hp, mp, fp, exp, name?, level?, titleText?, expExact?, expText?, hasBarNumbers, barTextPixels, toggledNumberClicks, latencyMs, box, titleBox, guideRect, searchRect, portrait, bars }`
- `nhk.setMapAnchor({ client, name?, region?, overlay?, overlayDurationMs?, quality?, silent? })` | `nhk.getMapDrift({ client, name?, region?, maxShift?, overlay?, overlayDurationMs?, quality?, silent? })` -> `{ ok, dx, dy, distance, returnAngle, confidence, monsterCount, monsterDots, cx, cy, radius, outerRadius, headingDeg?, turnDelta?, nearestMonsterAngle?, densestMonsterAngle?, latencyMs }` | `nhk.clearMapAnchor(client, name?)`
- `nhk.takeScreenshot({ client, prefix, region, subfolder, silent })` -> `filepath | nil`
- `nhk.takeDesktopScreenshot({ monitor, subfolder, prefix, region, silent })` -> `filepath | nil`
- `nhk.snapToRam({ client, bucket?, prefix?, region?, maxFrames?, maxMb?, useStream?, maxStreamAgeMs?, fallbackCapture?, quality?, silent?, log? })` -> `{ ok, bucket, seq, count, bytes, totalBytes, totalMb, dropped, source, latencyUs }` | `nhk.saveRamSnaps({ bucket?, subfolder?, prefix?, clearAfter?, async?, wait?, silent? })` | `nhk.getRamSnaps({ bucket? })` | `nhk.clearRamSnaps({ bucket?, silent? })`
- `nhk.captureFrame({ client?, monitor?, hwnd?, region? })` -> `{ id, width, height }` / `nhk.releaseFrame(id)` / `nhk.saveFrame({ frameId, path, region? })` *(RAM Frame Pool)*
- `nhk.getPixel({ frameId|client, x, y })` -> `{ r, g, b, a, hex }`
- `nhk.findColor({ frameId|client, color, tolerance?, region?, step?, maxResults? })` -> `{ found, count, x, y, matches }`
- `nhk.countColor({ frameId|client, color, tolerance?, region?, step? })` -> `{ count, total, ratio, percent, minX?, minY?, maxX?, maxY? }`
- `nhk.findRowBar({ frameId|client, region, color, tolerance? })` -> `{ percent, fillWidth, totalWidth, x, y }`
- `nhk.compareRegion({ frameId1, frameId2, region?, threshold?, step? })` -> `{ changed, diffCount, total, diffRatio, percent }`
- `nhk.findImage({ frameId|client, image, threshold?, region?, step?, mode? })` -> `{ found, x, y, confidence, width, height }` *(Supports local path, `http(s)://` URL with startup preload + RAM/disk cache, and `data:image/...;base64,...`)*
- `nhk.ocrRegion({ frameId|client, region?, lang? })` -> `{ text, lines }` *(Windows Native OCR)*
- **Custom HTML/CSS/JS Overlays:** `nhk.createOverlay(id, htmlOrOpts)` | `nhk.sendOverlay(id, event, data?)` | `nhk.closeOverlay(id)` | `nhk.onOverlayMessage(id, callback)` *(2-way `window.nhkOverlay` bridge)*
- **Canvas Drawing Overlays:** `nhk.drawRect({ client, x, y, w, h, color?, thickness?, fill?, label?, durationMs?, id? })` | `nhk.drawCircle({ client, x, y, r, color?, thickness?, fill?, label?, durationMs?, id? })` | `nhk.drawLine({ client, x1, y1, x2, y2, color?, thickness?, durationMs?, id? })` | `nhk.drawText({ client, x, y, text, color?, size?, bg?, durationMs?, id? })` | `nhk.clearDraw(client, id?)`

### Web DOM, Network & WASM (`web_dom`)
*(Note: `web*` targets HTML DOM on normal websites; use Vision Primitives or Opt-In WASM for `<canvas>`/`WebGL` games. `netStart` and `wasmEnable` are opt-in with zero browser-startup footprint.)*
- `nhk.webEval({ client, js })` | `nhk.webQuery({ client, selector, attr?, all? })` | `nhk.webWaitFor({ client, selector, timeoutMs? })` | `nhk.webClick({ client, selector })` | `nhk.webType({ client, selector, text, clear? })` | `nhk.webInfo({ client })` | `nhk.webNavigate({ client, url })`
- `nhk.netStart({ client })` | `nhk.netPoll({ client, limit? })` | `nhk.netStop({ client })` *(Buffers `ws_open`/`ws_recv`/`ws_send` CDP frames)*
- `nhk.wasmEnable({ client, reload? })` | `nhk.wasmDisable({ client })` | `nhk.wasmInfo({ client })` | `nhk.wasmScan({ client, value, type?, index?, offsets?, maxResults? })` | `nhk.wasmRead({ client, offset, type?, length?, index? })` *(Pass `offsets` for Next Scan; `type`: `i32`/`u32`/`i16`/`f32`/`f64`/`string`/`bytes`)*

### Desktop App UI Automation (`app_uia`)
- `nhk.appListWindows({ filter? })` -> `[{ hwnd, pid, title, processName, className, rect }]`
- `nhk.appReadText({ hwnd?, title?, process?, controlType?, name?, maxDepth? })` -> `[{ name, value, controlType, automationId, rect }]`
- `nhk.appClickControl({ hwnd?, title?, process?, name?, automationId?, controlType? })` -> `boolean`

### Utilities, HTTP, Clipboard, Storage & Safety
- `nhk.log(level, msg)` or `nhk.log({ message, level, silent })` — Levels: `"action"`, `"info"`, `"system"`, `"warn"`, `"error"`, `"log"`, `"success"`. (English ASCII only).
- `nhk.playSound({ sound, volume })`
- `nhk.speakTts({ text, voice, silent })` *(Pro)*
- `nhk.sendWebhook({ url, message, image })`
- `nhk.httpRequest({ url, method?, headers?, body?, timeoutMs? })` -> `{ status, ok, body, json?, error? }` *(Feature: `http_client`)*
- `nhk.getClipboard()` / `nhk.setClipboard({ text })`
- `nhk.writeFile({ path, content, append? })` | `nhk.readFile({ path })` | `nhk.writeJson({ path, data, pretty? })` | `nhk.readJson({ path })` *(Supports full paths & `%USERNAME%` / `~`)*
- `nhk.emergencyStop({ reason, scope, client, node_id, profile })`

---

## 3. External HTTP / REST / SSE Endpoints (`http://127.0.0.1:3088`)
- `GET/POST /api/webhook/{route}` — Directly triggers `nhk.onWebhook({ route = ... })` with JSON/text body.
- `GET/POST /api/trigger/{event_name}` — Triggers hotkey (`is_down` bool optional) + `"webhook"` + `"event"` flows.
- `POST /api/overlay/input` — Body `{ "name": "...", "value": ... }` updates registered overlay input & fires `onOverlayInput`.
- `GET /api/status` | `GET /api/active-clients` | `GET /api/client-frame/{id}`
- `POST /api/engine/start` | `POST /api/engine/stop` | `POST /api/suspend/toggle`
- `POST /api/client/launch` | `POST /api/client/close` | `POST /api/client/{id}/focus`
- `GET /api/signals/stream` | `GET /api/profile-events/stream` | `GET /api/profiles/stream` (SSE)

---

## 4. Visual Node Mode (`mode: "node"`) & Node Types
- **Graph JSON:** `{ "mode": "node", "nodes": [{ "id", "type", "title", "position": {"x","y"}, "data": {...} }], "connections": [{ "fromNodeId", "fromPort", "toNodeId", "toPort" }], "variables": [...] }`.
- **Connections:** Exec wires (`exec_out -> exec_in`, `true`/`false`, `on_tick`/`on_complete`) control flow; Data wires pass values or sync via `variables_map`. Node Mode and Code Mode communicate via `emit_event` ↔ `nhk.onEvent` and `nhk.emitEvent` ↔ `trigger` (`triggerType: "event"`).
- **Available Node `type` values:**
  - **Trigger:** `trigger` (`triggerType`: `"keyboard"|"mouse"|"event"|"webhook"`, `triggerValue`)
  - **Client:** `client_start`, `client_stop`, `client_check`
  - **Input:** `key_press`, `key_hold`, `forwarder`, `mouse_click`
  - **Flow/Logic:** `branch`, `var_branch`, `condition_group`, `wait` (`delay`), `timer`, `loop` (`repeat_loop`), `loop_scheduler`, `sequencer`, `debounce`, `reroute`, `control`
  - **Data/Utils:** `var_get`, `var_set` (`variable`), `format_text`, `message_log`, `sound`, `tts`, `screenshot`, `desktop_screenshot`, `webhook_out`, `emit_event`, `emergency_stop`
  - **Vision (Pro):** `party_scanner`, `party_heal`, `party_buff`, `party_slot`, `monster_scanner`, `monster_target`, `target_status`

