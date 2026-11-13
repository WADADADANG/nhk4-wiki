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

