# 07. Visual Node Mode & Node Reference

NHK4's **Visual Node Mode** (`mode: "node"`) uses an Unreal Engine Blueprint-style node graph stored as JSON (`configs/profiles/*.json`) and executed natively by `NodeExecutionEngine` + `ActionExecutor` in Rust.

---

## 1. How Node Connections Work

### Profile JSON Structure (`src/model.rs`)
```json
{
  "version": "4.0.0",
  "name": "My Workflow",
  "mode": "node",
  "nodes": [
    {
      "id": "node_1",
      "type": "trigger",
      "title": "F1 Hotkey",
      "position": { "x": 100, "y": 150 },
      "data": { "triggerType": "keyboard", "triggerValue": "F1" }
    },
    {
      "id": "node_2",
      "type": "key_press",
      "position": { "x": 400, "y": 150 },
      "data": { "targetClient": "1", "key": "1", "holdMs": 50 }
    }
  ],
  "connections": [
    {
      "fromNodeId": "node_1",
      "fromPort": "exec_out",
      "toNodeId": "node_2",
      "toPort": "exec_in"
    }
  ],
  "variables": []
}
```

### Pin Types
1. **Execution Pins (Flow Wires):** Control execution order from left to right.
   - Common input ports: `exec_in` (or `in`, `input`, `trigger_in` — matched case-insensitively).
   - Common output ports: `exec_out` (or `out`), plus conditional/branch outputs such as `true` / `false`, `on_tick` / `on_complete`, `on_found` / `on_empty`, `online` / `offline`.
2. **Data Pins (Value Wires):** Pass runtime values (e.g., `hp_out`, `slot_out`, `name_out`, `count`, `filepath`, `formatted_text`) between nodes or into `variables_map`.

### Connecting Visual Node Mode ↔ Luau Code Mode
- **Node → Code:** Use the `emit_event` node to broadcast a named event; any active Luau script listening with `nhk.onEvent({ event = "..." })` receives it immediately.
- **Code → Node:** Call `nhk.emitEvent({ event = "...", payload = ... })` in Luau; any active Visual Node profile with a matching `trigger` (`triggerType: "event"`, `triggerValue: "..."`) fires immediately.
- **Shared Variables:** `var_get` / `var_set` nodes and `nhk.getVar` / `nhk.setVar` read and write the same shared `engine.variables_map`.

---

## 2. Complete Node Reference (All Categories)

### A. Triggers (Entry Points)
| Node `type` | Description | Key `data` Fields |
| :--- | :--- | :--- |
| `trigger` | Entry point fired by keyboard hotkey (including combos like `ALT+1`), mouse button, custom event (`triggerType: "event"`), or inbound webhook (`triggerType: "webhook"`). | `triggerType` (`"keyboard"`, `"mouse"`, `"event"`, `"webhook"`), `triggerValue` (or `triggerKey`) |

---

### B. Client Management (`nodes/client/`)
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `client_start` | Launch Client | Launches the browser/game client in the specified fixed client slot (`1..N`). |
| `client_stop` | Stop / Close Client | Closes a running client slot (or multiple/`all`). |
| `client_check` | Check Client Online Status | Checks whether the target client slot is attached and responsive, branching on online vs. offline. |

---

### C. Input & Controls (`nodes/input/`, `nodes/window/`)
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `key_press` | Single Key Press | Dispatches a single simulated key press + release (`holdMs`) to the target client slot(s) via CDP (or OS). |
| `key_hold` | Key Hold Toggle | Holds, releases, or toggles a key continuously on target client(s). |
| `forwarder` | Multi-Client Forwarder | Broadcasts a physically held key in real time from the main window to target client slots (`1,2`, `all`). |
| `mouse_click` | Mouse Click | Dispatches a left/right/middle single or double click at the current cursor or configured coordinates. |

*(Additional window/mouse helpers in `nodes/window/` such as `focus_window`, `fake_focus`, `save_focus`, `restore_focus`, `focus_click`, `focus_move_click`, `focus_press_key`, `mouse_move`, `mouse_drag`, `mouse_move_click`, `get_window_info`, `is_window_focused` are exposed in **Code Mode** as `nhk.*` functions.)*

---

### D. Logic, Flow & Timers (`nodes/system/` & Core Flow)
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `branch` | Action Branch (If/Else) | Evaluates a condition/boolean and routes flow to `true` or `false` output pins. |
| `var_branch` | Variable Condition Branch | Compares a profile/global variable against a target value (`==`, `!=`, `>`, `<`, etc.) and branches `true`/`false`. |
| `condition_group` | Condition Group | Combines multiple conditions with `AND` / `OR` logic before branching. |
| `wait` *(alias `delay`)* | Wait / Sleep | Pauses workflow execution asynchronously for `ms` (non-blocking; aborts immediately on Engine Stop). |
| `timer` | Action Timer | Creates an interval timer (`interval` ms, repeat count or infinite) that fires its tick output without blocking. |
| `loop` *(alias `repeat_loop`)* | Repeat Loop | Repeats a sub-flow N times or continuously with a configured delay between iterations. |
| `loop_scheduler` | Loop Scheduler | Schedules periodic loop execution on a managed interval. |
| `sequencer` | Sequencer | Executes multiple output branches (`then_0`, `then_1`, ...) sequentially in order. |
| `debounce` *(alias `signal_filter`)* | Debounce (First & Last) | Filters rapid repeated trigger signals so only the leading/trailing signal passes within a cooldown window. |
| `reroute` | Reroute Knot | Pass-through wire knot used to keep complex Blueprint graphs clean and readable. |
| `control` *(alias `action_control`)* | Action Control | Controls/toggles execution state of target actions or loops. |

---

### E. Variables & Data (`nodes/system/`)
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `var_get` | Get Variable | Reads the current value of a named variable from `variables_map`. |
| `var_set` *(alias `variable`)* | Set Variable | Writes or updates a variable in `variables_map` (supports type coercion). |
| `format_text` | Format Text | Combines template strings and dynamic input values (e.g., `{name_out}`, `{hp_out}`) into a formatted string. |

---

### F. Safety, Alerts & Utilities (`nodes/system/`)
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `message_log` | Print Message Log | Outputs a formatted message to the NHK4 live log stream. |
| `sound` | Play Sound Alert | Plays a `.wav` / `.mp3` alert sound from `public/sounds/`. |
| `tts` | Text to Speech (TTS) *(Pro)* | Synthesizes and speaks text via Edge TTS asynchronously. |
| `screenshot` | Capture Client Screenshot *(Pro)* | Captures a zero-flicker CDP screenshot of a client slot and outputs the saved file path. |
| `desktop_screenshot` | Capture Desktop Screenshot *(Pro)* | Captures a Windows monitor screenshot and saves the file path. |
| `webhook_out` | Send Webhook Alert | Sends an HTTP POST message (and optional attached screenshot) to Discord or any webhook URL. |
| `emit_event` | Broadcast Event | Emits a named custom event (with optional payload) to other Node workflows and Luau Code scripts. |
| `emergency_stop` | Emergency Stop All | Immediately halts all running engines/workflows (or a scoped target). |

---

### G. Vision & Party (`nodes/vision/`) *(Pro)*
| Node `type` | Title | Description |
| :--- | :--- | :--- |
| `party_scanner` | Scan Party Health | Scans the Party HUD (`scanRegion: "right"`), detects all 2–8 members, computes `lowestHp` / `lowestSlot` / `hasLowHp`, and renders optional debug overlay. |
| `party_heal` | Party Auto-Heal | Finds the lowest-HP alive/in-range member below `threshold`, clicks them, and presses the heal key. |
| `party_buff` | Party Buff Cycle | Rotates through alive, in-range, clickable party members one by one per trigger and casts the buff key. |
| `party_slot` | Select Party Slot | Fresh-scans the party list, clicks the exact crimson bar center of slot `1..8`, and outputs `name_out`, `slot_out`, `hp_out`. |
| `monster_scanner` | Scan Monsters on Screen | Scans visible monster nameplates on screen (`aggro_first`, `nearest`, name filter) and outputs counts + best target coordinates. |
| `monster_target` | Target & Attack Monster | Scans for the best matching monster, clicks it, and optionally presses the attack key. |
| `target_status` | Get Target Status | Reads the top selected-target bar HUD (`hasTarget`, `hp`, `isDead`, `isMonster`, `className`). |
