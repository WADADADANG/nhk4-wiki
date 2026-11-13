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

## 3. Monster Scanner & Targeting (`vision_monster.rs`)

### `nhk.scanMonsters(opts | client, [name], [priority], [overlay])`
### `nhk.scanMonstersStream(opts | client, [name], [priority], [overlay])` *(RAM Stream <10ms)*
Scans visible monster nameplates on screen.
- **Params:**
  - `client` (number)
  - `name` (or `monster`, string filter, default `""` = all)
  - `priority` (or `mode`, default `"aggro_first"`)
  - `offsetY` (number, default `35` px below nameplate for click point)
  - `maxDistance` (or `maxDist`, number, default `0` = unlimited)
  - `minScore` (number, default `0.52`)
  - `overlay` (or `showOverlay`, default `true`)
  - `quality` (number `20..100`, default `65`)
  - `silent` (bool)
- **Returns Table:**
  - `count`, `aggroCount`, `passiveCount`
  - `hasTarget` (boolean), `targetHp` (number)
  - `latencyMs` (number, on `scanMonstersStream`)
  - `best` (table | nil): Best matching monster `{ x, y, clickX, clickY, isAggro, distance, dist, score, trackId, id }`
  - `monsters` (array of monster tables)

### `nhk.targetMonster(opts | client, [name], [attackKey], [priority], [offsetY], [overlay])`
Scans and clicks the best monster (if `click = true`, default `true`) and optionally presses `attackKey`.
- **Extra Params:** `attackKey` (or `key`), `click` (or `clickTarget`, default `true`), `delayAfterClick` (default `70` ms)
- **Returns Table:** `{ found, success, count, x, y, clickX, clickY, isAggro, distance, dist, score, trackId, id }`

---

## 4. Target Status Bar (`vision_target.rs`)

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

---

## 5. Screenshots (`vision.rs`)

### `nhk.takeScreenshot(opts | client, [prefix], [region])`
Captures a client CDP screenshot and saves it under `./screenshots/{subfolder}/`.
- **Params:** `client`, `prefix` (or `filename`, `name`, default `"snap"`), `region` (`"full"` default, or `{ x, y, width, height }`), `subfolder` (or `folder`, default `"client_<id>"`), `silent` (bool)
- **Returns:** `filepath` (string) or `nil`

### `nhk.takeDesktopScreenshot(opts | monitor, [subfolder], [prefix], [region])`
Captures an OS monitor screenshot and saves its path to `last_screenshot` variable.
- **Params:** `monitor` (default `1`), `subfolder` (default `"desktop"`), `prefix` (default `"snap"`), `region` (default `"full"`), `silent` (bool)
- **Returns:** `filepath` (string) or `nil`

