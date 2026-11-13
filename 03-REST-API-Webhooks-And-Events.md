# 03. REST API, Webhooks & External Events

NHK4 runs a local HTTP & Server-Sent Events (SSE) server on **`http://127.0.0.1:3088`**. External scripts (Python, Node.js, AutoHotkey, Stream Deck, OBS, Discord bots, curl) can control NHK4, trigger Luau callbacks, update overlay inputs, and subscribe to live status/events.

---

## 1. Triggering Events & Inbound Webhooks

### Trigger Hotkey / Event / Webhook
- **Endpoint:** `GET` or `POST /api/trigger/{event_name}`
- **Behavior:**
  1. Processes `{event_name}` as a hotkey trigger (if payload contains `{"is_down": true|false}`, dispatches key down/up; otherwise fires a 40ms pulse).
  2. Dispatches both `"webhook"` and `"event"` flows with the optional JSON or plain-text request body as payload.
- **Response:** `{ "success": true, "triggered": "<event_name>" }`

### Direct Inbound Webhook
- **Endpoint:** `GET` or `POST /api/webhook/{route}`
- **Behavior:** Directly invokes matching `nhk.onWebhook({ route = "..." })` listeners in Luau Code Mode and Webhook Trigger nodes in Visual Node Mode.
- **Example (curl):**
```bash
curl -X POST http://127.0.0.1:3088/api/webhook/heal-now \
  -H "Content-Type: application/json" \
  -d '{"client": 2, "reason": "low_hp"}'
```
- **Matching Luau Handler:**
```lua
nhk.onWebhook({
    route = "heal-now",
    callback = function(payload)
        local cid = (payload and payload.client) or 2
        nhk.pressKey({ client = cid, key = "F1" })
    end
})
```

---

## 2. Outbound Webhooks (From NHK4 to External Services)
Use `nhk.sendWebhook` inside Luau to POST JSON, plain text (`content`), or multipart image files (e.g., Discord webhooks):
```lua
local imgPath = nhk.takeScreenshot({ client = 1, prefix = "boss" })
nhk.sendWebhook({
    url = "https://discord.com/api/webhooks/...",
    message = "Boss spawned on Client 1!",
    image = imgPath
})
```
*Note: If `message` starts with `{`, NHK4 parses/sends it as raw JSON (or `payload_json` when attached with an image).*

---

## 3. Engine, Client & Overlay Control Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/status` | Full engine status, running state, active profiles, clients, overlay inputs |
| `GET` | `/api/active-clients` | List of active client slots |
| `POST` | `/api/engine/start` | Start the automation engine |
| `POST` | `/api/engine/stop` | Stop the automation engine |
| `POST` | `/api/suspend/toggle` | Toggle suspend/resume state |
| `POST` | `/api/overlay/disable` | Disable overlay |
| `POST` | `/api/overlay/input` | Update a registered Overlay Input value (`{ "name": "...", "value": ... }`) |
| `POST` | `/api/client/launch` | Launch a client slot |
| `POST` | `/api/client/close` | Close a client slot |
| `POST` | `/api/client/focus` | Focus a client window |
| `POST` | `/api/client/{id}/focus` | Focus specific client slot `{id}` |
| `GET` | `/api/client-frame/{id}` | Latest JPEG frame from client `{id}` stream |

---

## 4. Server-Sent Events (SSE) Streams

| Method | Endpoint | Purpose |
| :--- | :--- | :--- |
| `GET` | `/api/signals/stream` | Real-time engine status, logs, and variable/overlay updates |
| `GET` | `/api/profile-events/stream` | Profile event notifications |
| `GET` | `/api/profiles/stream` | Profile list/update stream |

---

## 5. Cross-Profile Communication (Internal Events)
Active Luau profiles and Visual Node profiles can communicate with zero polling using `nhk.emitEvent` and `nhk.onEvent`:
```lua
-- Profile A (Sender)
nhk.emitEvent({
    event = "BOSS_ALERT",
    payload = { client = 1, hp = 25 }
})

-- Profile B (Receiver)
nhk.onEvent({
    event = "BOSS_ALERT",
    callback = function(payload)
        nhk.log("info", "Boss alert from client " .. tostring(payload.client))
    end
})
```
*Note: Event names in `onEvent` are normalized to uppercase internally (`event_name.trim().to_uppercase()`).*

