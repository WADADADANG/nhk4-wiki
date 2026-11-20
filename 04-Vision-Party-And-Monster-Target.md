# 04. Vision, Party, Monster & Target APIs

All Vision APIs (except `startStream` / `stopStream`) require a **Pro license** and target a running **Client Slot** (`client = 1`, etc.).

---

## 1. Screencast RAM Stream (Ultra-Fast <10ms Vision)
Standard vision calls (`scanParty`, `scanMonsters`, `getTarget`) capture a fresh CDP screenshot per call (~30-60ms).
For high-frequency combat loops, start a background screencast stream into RAM (`pip_frame_cache`) and use the `*Stream` functions (<5-10ms latency):

```lua
nhk.startStream({ client = 1, quality = 65 })
-- Later when done:
nhk.stopStream({ client = 1 })
```
- **`nhk.startStream(opts | client, [quality])`:** `quality` clamped `20..100` (default `65`). Returns `boolean`.
- **`nhk.stopStream(opts | client)`:** Stops screencast and clears cached frame. Returns `boolean`.

---

## 2. Party List Vision & Support (`vision.rs`)

### `nhk.scanParty(opts | client, [region], [threshold], [overlay])`
Scans the Flyff Party HUD (default `region = "right"`).
- **Params:** `client`, `region` (or `mode`, default `"right"`), `threshold` (or `hpThreshold`, default `70.0`), `overlay` (or `showOverlay`, default `true`), `silent` (bool)
- **Returns Table:**
  - `count` (number): Total detected party members
  - `aliveCount` (number): Alive members
  - `totalSlots` (number)
  - `lowestHp` (number): Lowest HP % (`0..100`)
  - `hasLowHp` (boolean): `true` if any alive member has `hp <= threshold`
  - `lowestSlot` (number | nil): Slot `1..8` with lowest HP
  - `members` (table): Map `[slot] = hp_percent`
  - `details` (table): Map `[slot] = { slot, hp, isAlive, isLowHp, inRange, isLeader, isDead, isOffline, clickable, x, y, width, height }`

### `nhk.partyHeal(opts | client, [threshold], [healKey], [region], [clickTarget], [showOverlay], [delayAfterClick])`
Finds the lowest-HP alive/in-range party member below `threshold`, clicks them (if `clickTarget = true`), and presses `healKey`.
- **Defaults:** `threshold = 70.0`, `region = "right"`, `clickTarget = true`, `showOverlay = true`, `delayAfterClick = 80` ms.
- **Returns:** `boolean` (`true` if heal was triggered).

### `nhk.partyBuffNext(opts | client, [buffKey], [region])`
Rotates through alive, in-range, clickable party members one slot per call, clicks the member, and presses `buffKey`.
- **Returns:** `boolean` (`true` if a member was buffed, `false` when rotation completes a cycle or no active members).

### `nhk.selectPartySlot(opts | client, slot, [region])`
Fresh-captures the party list, clicks the exact crimson bar center of party `slot` (`1..8`), moves cursor slightly away to avoid tooltip occlusion, and updates `name_out`, `slot_out`, `hp_out` in `variables_map`.
- **Params:** `client`, `slot` (or `targetSlot`: `1..8`), `region` (default `"right"`), `overlay` (default `true`), `delay` (or `delayAfterClick`, default `80`)
- **Returns:** `number` (selected slot, or `0` on failure).

---

## 3. Monster Scanner, Body Finder & Hover Verification (`vision/monster.rs`, `monster_body.rs`, `hover.rs`)

### `nhk.scanMonsters(opts | client, [name], [priority], [overlay])`
### `nhk.scanMonstersStream(opts | client, [name], [priority], [overlay])` *(RAM Stream <10ms)*
Scans visible monster nameplates on screen. When `name` is provided (non-empty and not `"*"`), each candidate nameplate is **verified using real Windows OCR** (cached 15s per `trackId`, never guessed from nameplate width). Uncached **Red (Aggro)** nameplates automatically trigger a tiny event-driven **lossless PNG clip** (~3-4 KB, once per `trackId`) so JPEG 4:2:0 chroma subsampling never blurs red text before OCR.
- **Params:**
  - `client` (number)
  - `name` (or `monster`, string comma-separated filter e.g. `"SmallSamoset"` or `"SmallSamoset,Aibatt"`, default `""` = all)
  - `priority` (or `mode`, default `"aggro_first"`)
  - `offsetY` (number, default `35` px below nameplate for click point)
  - `maxDistance` (or `maxDist`, number, default `0` = unlimited)
  - `minScore` (number, default `0.52`, clamped `0.50..0.95` for OCR similarity)
  - `skipCenter` (or `excludeCenter`, `ignoreCenter`, bool, default `true` — set `false` when melee mobs overlap center)
  - `skipPlayers` (or `excludePlayers`, `ignorePlayers`, bool, default `true` — filters player/guild nameplates)
  - `overlay` (or `showOverlay`, default `true`)
  - `quality` (number `20..100`, default `65`)
  - `silent` (bool)
- **Returns Table:**
  - `count`, `aggroCount`, `passiveCount`
  - `hasTarget` (boolean), `targetHp` (number), `isMonsterTarget` (boolean)
  - `latencyMs` (number, on `scanMonstersStream`)
  - `best` (table | nil): Best matching monster `{ index, name, x, y, width, height, clickX, clickY, isAggro, distance, dist, score, pixelCount, trackId, id }`
  - `monsters` (array of monster tables)

### `nhk.targetMonster(opts | client, [name], [attackKey], [priority], [offsetY], [overlay])`
Scans and clicks the best monster (if `click = true`, default `true`) and optionally presses `attackKey`.
- **Extra Params:** `attackKey` (or `key`), `skipCenter`, `skipPlayers`, `click` (or `clickTarget`, default `true`), `delayAfterClick` (default `70` ms)
- **Returns Table:** `{ found, success, count, aggroCount, passiveCount, hasTarget, targetHp, name, x, y, clickX, clickY, isAggro, distance, dist, score, trackId, id }`

### `nhk.findMonsterBody(opts)` & `nhk.findMonsterBodies(client, mobsList, [opts])` *(Feature: `monster_scanner`)*
Finds the actual clickable 3D monster body beneath a detected nameplate (using perspective ROI + foreground segmentation + optional 2-frame motion boost) so clicks hit the monster's torso instead of empty ground.
- **Params (`findMonsterBody`):**
  - `client` (number), plus `mob` (monster table from `scanMonsters`) OR raw `x, y, width (w), height (h)`
  - `padX`, `offsetY`, `scanHeight` (or `bodyHeight`), `motionWeight` (default `1.2`), `useMotion` (bool, default `true`)
  - `maskPlayer` (or `skipCenter`, `excludeCenter`, bool, default `true` — set `false` if other players are hidden so melee center mobs are never masked)
  - `maxCandidates` (`1..6`, default `3`), `overlay` (or `showOverlay`, bool, default `false`), `overlayDurationMs` (default `2500`), `quality` (default `85`), `silent` (bool)
- **Returns (`findMonsterBody`):** `{ found, clickX, clickY, areaPixels, coreDepth, confidence, latencyMs, bodyBox = { x, y, width, height }, roiBox = { x, y, width, height }, candidates = [{ x, y, coreDepth }, ...] }`
- **`findMonsterBodies(client, mobsList, opts?)`:** Processes an entire array of monsters from **one single frame capture** and returns an array of body result tables.

### `nhk.isMonsterHovered(opts)` *(Feature: `hover_scan`)*
Detects the bright hover selection outline around a monster after `nhk.mouseMove(...)` before committing to a left click.
- **Params:**
  - `client` (number), plus `mob` (monster table) OR `x, y, width, height`
  - `padX` (default `25`), `offsetY` (default `6`), `scanHeight` (or `bodyHeight`, default `110`), `minPixels` (default `25`)
  - `fresh` (bool, default `false` — set `true` to bypass RAM stream and force a fresh post-move capture)
  - `maxFrameAgeMs` (`10..500`, default `60` — rejects stale pre-`mouseMove` stream frames)
  - `quality` (default `85`), `silent` (bool)
- **Returns:** `{ hovered, outlinePixels, latencyMs, box = { x, y, width, height } }`

---

## 4. Target Status Bar, Character Status UI & Minimap Navigation (`target.rs`, `char_status.rs`, `minimap.rs`)

### `nhk.getTarget(opts | client)`
### `nhk.getTargetStream(opts | client)` *(RAM Stream <5ms)*
Reads the top target status bar HUD.
- **Params:** `client`, `log` (bool, default `false`), `overlay` (or `showOverlay`, default `false`), `quality` (default `60`, `getTarget` only)
- **Returns Table:**
  - `hasTarget` (boolean)
  - `className` (string, or `"none"`)
  - `hp` (number `0..100`)
  - `isDead` (boolean)
  - `isMonster` (boolean)
  - `hasNumbers` (boolean)
  - `hasMpBar` (boolean)
  - `circleType` (string)
  - `scale` (number)
  - `hasGoldX` (boolean)
  - `x`, `y`, `width` (numbers)
  - `latencyMs` (number)

### `nhk.scanCharStatus(opts | client, [position | opts])`
### `nhk.scanCharStatusStream(opts | client, [position | opts])` *(RAM Stream)*
Scans the player's own **Character Status Window** (supports arbitrary UI scales `0.35x..2.5x` and all 3 bar-number display modes).
- **Params:**
  - `client` (number)
  - `position` (or `zone`, `anchor`, e.g. `"top_left"`, `"auto"`)
  - `scale` (or `uiScale`, optional fixed scale), `minScale`, `maxScale`, `region` (`{ x, y, width|w, height|h }`, optional custom ROI)
  - `ocr` (or `readText`: `"auto"` default, `true`/`"always"`, `false`/`"never"` — `"auto"` caches name/level/title and re-runs OCR automatically on window move or level-up)
  - `ocrLang` (or `lang`, default `"en-US"`), `autoHideNumbers` (or `hideNumbers`, bool, default `true`), `useStream` (bool)
  - `overlay` (bool, default `false`), `guide` (or `showGuide`, bool, default `true` — shows placement box when HUD is not found), `overlayDurationMs` (default `2500`), `quality` (default `85`), `silent` (bool)
- **Returns Table:**
  - `found`, `zone`, `scale`, `confidence`, `hp`, `mp`, `fp`, `exp` (`0..100`)
  - `name?`, `level?`, `titleText?`, `expExact?`, `expText?`
  - `hasBarNumbers`, `barTextPixels`, `toggledNumberClicks`, `latencyMs`
  - `box`, `titleBox`, `guideRect`, `searchRect`, `portrait = { x, y, width, height, centerX, centerY, hasPortrait }`
  - `bars = { hp, mp, fp, exp }` (each `{ percent, x, y, width, height, centerX, centerY, filledWidth, hasText, textPixels }`)

### Minimap Anchor & Drift Navigation (`minimap.rs`)
Tracks player drift from a farming spot anchor and locates dense orange monster clusters on the top-right circular minimap.
- **`nhk.setMapAnchor({ client, name?, region?, overlay?, overlayDurationMs?, quality?, silent? })`** (or `nhk.setMapAnchor(client, opts?)`): Captures the current minimap terrain as the reference anchor.
- **`nhk.getMapDrift({ client, name?, region?, maxShift?, overlay?, overlayDurationMs?, quality?, silent? })`** (or `nhk.getMapDrift(client, opts?)`):
  - **Returns:** `{ ok, dx, dy, distance, returnAngle, confidence, monsterCount, monsterDots, cx, cy, radius, outerRadius, latencyMs, headingDeg?, turnDelta?, nearestMonsterAngle?, densestMonsterAngle?, error? }`
- **`nhk.clearMapAnchor(client, [name])`:** Clears stored anchor(s) and removes the minimap overlay. Returns removed count.

---

## 5. Screenshots & In-RAM Debug Snapshot Buffer (`screenshot.rs`, `ram_buffer.rs`)

### `nhk.takeScreenshot(opts | client, [prefix], [region])`
Captures a client CDP screenshot and saves it under `./screenshots/{subfolder}/`.
- **Params:** `client`, `prefix` (or `filename`, `name`, default `"snap"`), `region` (`"full"` default, or `{ x, y, width, height }`), `subfolder` (or `folder`, default `"client_<id>"`), `silent` (bool)
- **Returns:** `filepath` (string) or `nil`

### `nhk.takeDesktopScreenshot(opts | monitor, [subfolder], [prefix], [region])`
Captures an OS monitor screenshot and saves its path to `last_screenshot` variable.
- **Params:** `monitor` (default `1`), `subfolder` (default `"desktop"`), `prefix` (default `"snap"`), `region` (default `"full"`), `silent` (bool)
- **Returns:** `filepath` (string) or `nil`

### In-RAM Ring Buffer Snapshots (`ram_buffer.rs`) *(Feature: `screenshot`)*
Zero-lag, zero-disk-I/O event-driven screenshot recording into RAM during combat/debugging, flushed to disk only on demand.
- **`nhk.snapToRam({ client, bucket?, prefix?, region?, maxFrames?, maxMb?, useStream?, maxStreamAgeMs?, fallbackCapture?, quality?, silent?, log? })`** *(or positional `client, bucket?, prefix?`)*
  - Copies compressed JPEG directly from the screencast RAM cache (`<1ms`) or falls back to CDP capture. Default `silent = true`.
  - **Returns:** `{ ok, bucket, seq, count, bytes, totalBytes, totalMb, dropped, source, latencyUs }`
- **`nhk.saveRamSnaps({ bucket?, subfolder?, prefix?, clearAfter?, async?, wait?, silent? })`** *(or positional `bucket?, subfolder?`)*
  - Flushes buffered RAM frames to `./screenshots/{subfolder}/`. Defaults `clearAfter = true`, `async = true` (set `wait = true` for synchronous stats).
  - **Returns:** `{ ok, bucket, queued, saved, failed, totalBytes, totalMb, dir, files, elapsedMs, background }`
- **`nhk.getRamSnaps({ bucket? })`** *(or `bucket?`)* -> `{ bucket, count, totalBytes, totalMb, seq, buckets = [{ bucket, count, totalBytes, totalMb }, ...] }`
- **`nhk.clearRamSnaps({ bucket?, silent? })`** *(or `bucket?`)* -> clears RAM frames for a bucket (or all buckets).

---

## 6. Primitive Vision, Template Matching & Native OCR (`vision/frame.rs`, `pixel.rs`, `template.rs`, `ocr.rs`)

Use these **atomic primitives** to build custom scanners for **any game, UI, or web canvas** without relying on Flyff-specific HUD assumptions.

> **Critical Performance & Coordinate Rule for External AIs:**
> 1. **Capture once, scan many times:** Call `nhk.captureFrame(...)` once, pass `frameId = frame.id` to `getPixel` / `findColor` / `countColor` / `findRowBar` / `findImage` / `ocrRegion` (`< 1 ms` per scan in Rust), and always call `nhk.releaseFrame(frame.id)` when done.
> 2. **Relative coordinates on cropped frames:** If `captureFrame` was called with `region = { x = 100, y = 200, w = 220, h = 40 }`, the stored frame is `220x40` starting at `(0, 0)`. Subsequent scans on `frame.id` must use `region = { x = 0, y = 0, w = 220, h = 40 }` (not `(100, 200)`).

### RAM Frame Pool (`frame.rs`) *(Feature: `vision_primitives`)*
- **`nhk.captureFrame({ client?, monitor?, hwnd?, region? })` -> `{ id, width, height } | nil`**
  Captures a browser client (CDP full viewport cropped in Rust for zero WebGL flicker), Windows `hwnd`, or desktop `monitor` into the in-RAM `FramePool` (60s TTL, max 64 frames, zero disk I/O).
- **`nhk.releaseFrame(frameId)` -> `boolean`**
  Immediately frees a cached frame from RAM. Returns `true` if removed, `false` if already expired/missing.
- **`nhk.saveFrame({ frameId, path, region? })` -> `{ ok, path, width, height } | nil`**
  Saves a cached frame (or cropped `region`) to disk as `.png`. Supports full paths and `%USERNAME%` / `~` expansion (use `/` in Luau strings).

### Pixel & Region Primitives (`pixel.rs`) *(Feature: `vision_primitives`)*
- **`nhk.getPixel({ frameId? | client? | monitor? | hwnd?, x, y })` -> `{ r, g, b, a, hex } | nil`**
  Reads a single pixel's RGBA values and uppercase `#RRGGBB` hex string. Avoid calling in tight Luau loops; use `findColor` / `countColor` / `findRowBar` instead.
- **`nhk.findColor({ frameId? | client?, color, tolerance?, region?, step?, maxResults? })` -> `{ found, count, x?, y?, matches }`**
  Finds pixels matching `color` (`"#RRGGBB"` or `{ r, g, b }`) within per-channel `tolerance` (`0..255`, default `0`). `step` skips pixels for speed; `maxResults` defaults to `32` (max `500`). Returns first match in `x, y` and all matches in `matches` (`[{ x, y }, ...]`).
- **`nhk.countColor({ frameId? | client?, color, tolerance?, region?, step? })` -> `{ count, total, ratio, percent, minX?, minY?, maxX?, maxY? }`**
  Counts matching pixels in `region`, returning fill `percent` (`0..100`) and bounding box (`minX..maxY`). Useful for detecting buff icons, cooldown overlays, or status indicators.
- **`nhk.findRowBar({ frameId? | client?, region, color, tolerance? })` -> `{ percent, fillWidth, totalWidth, x, y }`**
  Finds the horizontal row inside `region` with the longest run of pixels matching `color` (`tolerance` default `15`) and computes fill percentage (`0..100`). Ideal for custom HP/MP/EXP/Cast bars.
- **`nhk.compareRegion({ frameId1, frameId2, region?, threshold?, step? })` -> `{ changed, diffCount, total, diffRatio, percent }`**
  Compares two cached frames pixel-by-pixel (`threshold` default `15`) to detect screen freezes, popups, damage flashes, or animations.

### Template Matching (`template.rs`) *(Feature: `vision_primitives`)*
- **`nhk.findImage({ frameId? | client? | monitor? | hwnd?, image, threshold?, region?, step?, mode? })` -> `{ found, x, y, confidence, width, height } | nil`**
  Searches for a `.png`/`.jpg` template image using raw-slice RGB SAD matching. Supports transparent PNG masks (`alpha >= 128`), `threshold` (`0.0..1.0`, default `0.88`), and `image` sources:
  - **Local path:** Full or relative file path (supports `%ENV_VAR%` / `~`).
  - **HTTP/HTTPS URL (`http://` / `https://`):** Automatically background-preloaded when the `.luau` profile loads, deduplicated in-flight, and cached in both RAM (`0ms` reuse) and disk (`%LOCALAPPDATA%\NHK4\cache\templates\`).
  - **Base64 Data URI (`data:image/...;base64,...`):** Lossless embedded image decoded once and cached in RAM.
  - `mode = "fast"` *(default)*: 16-point fast-reject pre-check + coarse stride + local 1px refinement around the best match.
  - `mode = "exact"`: Exhaustive scan at the exact `step` without pre-check skipping (best for tiny icons).

### Windows Native OCR (`ocr.rs`) *(Feature: `ocr_pixel`)*
- **`nhk.ocrRegion({ frameId? | client? | monitor? | hwnd?, region?, lang? })` -> `{ text, lines } | nil`**
  Recognizes text inside a frame or region using the built-in Windows 10/11 `OcrEngine` (no external Tesseract DLLs or cloud APIs). `lang` is an optional BCP-47 tag (e.g. `"en-US"`, `"th"`). Returns full `text` and `lines` (`[{ text, words = [{ text, x, y, w, h }] }]`).

```lua
-- Example: Read custom HP bar + numbers with 1 capture
local f = nhk.captureFrame({ client = 1, region = { x = 40, y = 40, w = 220, h = 50 } })
if f then
    local bar = nhk.findRowBar({ frameId = f.id, region = { x = 0, y = 0, w = 220, h = 20 }, color = "#DC2626", tolerance = 25 })
    local ocr = nhk.ocrRegion({ frameId = f.id, region = { x = 0, y = 20, w = 220, h = 30 }, lang = "en-US" })
    nhk.log(string.format("HP=%.1f%% Text=%s", bar.percent, ocr and ocr.text or ""))
    nhk.releaseFrame(f.id)
end
```


