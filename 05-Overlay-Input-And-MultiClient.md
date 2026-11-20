# 05. Overlay Input & Multi-Client Control

## 1. Dynamic Overlay UI Inputs
Luau scripts can expose live interactive controls (sliders/numbers, toggles/booleans, text inputs) directly in the NHK4 Overlay UI without hardcoding values.

### Registering a Control (`nhk.registerOverlayInput`)
```lua
nhk.registerOverlayInput({
    name = "heal_threshold",
    text = "Heal HP %",
    type = "number",
    default = 70,
    min = 10,
    max = 95,
    step = 5,
    client = 2,
    onChange = function(val)
        nhk.log("info", "Heal threshold updated to " .. tostring(val))
    end
})
```
- **Supported `type` values:** `"number"`, `"boolean"` / `"bool"`, `"string"` / `"text"`.
- **Persistence:** Values are stored in `engine.variables_map[name]`. If a value already exists in `variables_map`, `registerOverlayInput` preserves it instead of overwriting with `default`.
- **External Updates:** Can also be updated via HTTP `POST http://127.0.0.1:3088/api/overlay/input` with `{ "name": "heal_threshold", "value": 80 }`.

### Subscribing Separately (`nhk.onOverlayInput`)
```lua
nhk.onOverlayInput("heal_threshold", function(val)
    -- Called immediately with current value, and whenever user changes it
end)
```

---

## 2. Custom Status Badges on Overlay (`setStatusOverlay`)
Display live bot state (e.g., `"Buffing"`, `"Healing"`, `"Idle"`) on the client's overlay card:
```lua
nhk.setStatusOverlay({
    client = 2,
    name = "rm_bot",
    status = "Active",
    detail = "Scanning Party HP",
    type = "loop"
})

-- Clear later:
nhk.clearStatusOverlay({ client = 2, names = { "rm_bot" } })
-- Or clear all overlays across all clients:
nhk.clearStatusOverlay()
```

---

## 3. Multi-Client Window & Focus Helpers
- **Check if client is online:** `nhk.isClientOnline(1)`
- **Get window geometry & focus state:**
```lua
local info = nhk.getWindowInfo(1)
if info.online and not info.isMinimized then
    nhk.log("info", string.format("Client 1 size: %dx%d", info.clientWidth, info.clientHeight))
end
```
- **Save & restore active foreground window:**
```lua
local prev = nhk.saveFocus()
nhk.focusWindow(2)
nhk.wait({ ms = 50 })
nhk.restoreFocus(prev)
```
- **Background Fake Focus (`nhk.fakeFocus(slot)`):** Sends a fake focus message to the target window handle without stealing OS foreground focus.
- **Focus Click (`nhk.focusClick`):** Focuses the window, sends an OS click, and restores the previous foreground window + cursor position automatically (`restoreFocus = true` by default).

---

## 4. In-Game Canvas Debug Drawing Overlay (`web/draw.rs`)
Draw lightweight non-blocking vector shapes directly on top of a client viewport (click-through, auto-expiring via `durationMs`, or persistent by `id`).
- **`nhk.drawRect({ client, x, y, w|width, h|height, color?, thickness?, fill?, label?, durationMs?, id? })`** -> `boolean`
- **`nhk.drawCircle({ client, x, y, r|radius, color?, thickness?, fill?, label?, durationMs?, id? })`** -> `boolean`
- **`nhk.drawLine({ client, x1, y1, x2, y2, color?, thickness?, durationMs?, id? })`** -> `boolean`
- **`nhk.drawText({ client, x, y, text, color?, size?, bg?, durationMs?, id? })`** -> `boolean`
- **`nhk.clearDraw(client, [id])`** or **`nhk.clearDraw({ client, id? })`** -> `boolean` *(Omit `id` to clear all shapes on that client)*

```lua
nhk.drawRect({ client = 1, id = "mob_box", x = 770, y = 677, w = 80, h = 10, color = "#22c55e", label = "Small Samoset", durationMs = 2000 })
nhk.drawCircle({ client = 1, id = "click_pt", x = 810, y = 720, r = 8, color = "#38bdf8", durationMs = 2000 })
```

