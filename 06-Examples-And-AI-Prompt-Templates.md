# 06. Examples & AI Prompt Templates

## 1. Ready-to-Use Luau Script Examples

### Example A: Event-Driven Auto-Heal & Buff Toggle (Using Overlay Input + `setTimer`)
```lua
local HEALER_CLIENT = 2
local HEAL_KEY = "F1"

nhk.registerOverlayInput({
    name = "auto_heal_on",
    text = "Auto Heal",
    type = "boolean",
    default = false,
    client = HEALER_CLIENT,
    onChange = function(enabled)
        if enabled then
            nhk.setStatusOverlay({
                client = HEALER_CLIENT,
                name = "auto_heal",
                status = "Healing",
                detail = "Threshold 70%",
                type = "loop"
            })
            nhk.setTimer({
                name = "heal_timer",
                interval = 350,
                times = 0,
                callback = function()
                    nhk.partyHeal({
                        client = HEALER_CLIENT,
                        threshold = 70,
                        key = HEAL_KEY,
                        region = "right",
                        silent = true
                    })
                end
            })
        else
            nhk.killTimer("heal_timer")
            nhk.clearStatusOverlay({ client = HEALER_CLIENT, names = { "auto_heal" } })
        end
    end
})
```

### Example B: Stream Auto-Hunt & Combat State Machine (Production Pattern)
```lua
local CLIENT = 1
local MOB_NAME = "SmallHundurSharpfoot"
local ATTACK_KEY = "7"
local isRunning = false

-- Use "home" (or "insert"/"delete") for bot toggle; avoid F1-F12 (reserved for in-game skill bars)
nhk.onHotkey("home", function()
    isRunning = not isRunning

    if not isRunning then
        nhk.stopStream({ client = CLIENT })
        nhk.clearStatusOverlay({ client = CLIENT, names = { "auto_hunt" } })
        nhk.log("info", "Auto Hunt: STOPPED (Stream Closed)")
        return
    end

    nhk.log("info", "Auto Hunt: STARTED (Stream Mode)")
    nhk.startStream({ client = CLIENT, quality = 65 })
    nhk.sleep(200) -- Wait for first frame in RAM

    nhk.setStatusOverlay({
        client = CLIENT,
        name = "auto_hunt",
        status = "Auto Hunt",
        detail = "Searching " .. MOB_NAME,
        type = "loop"
    })

    while isRunning do
        -- 1. Scan nearest monster from RAM stream (<10ms)
        local scan = nhk.scanMonstersStream({
            client = CLIENT,
            name = MOB_NAME,
            priority = "nearest",
            maxDistance = 450,
            offsetY = 0,
            overlay = true,
            silent = true
        })

        if scan and #scan.monsters > 0 then
            local targetMob = scan.best or scan.monsters[1]
            local clickX = targetMob.clickX or (targetMob.x + math.floor(targetMob.width / 2))
            local clickY = targetMob.clickY or (targetMob.y + math.floor(targetMob.height / 2))

            -- 2. Click monster to select target
            nhk.mouseClick({
                client = CLIENT,
                button = "left",
                action = "single",
                x = clickX,
                y = clickY,
                overlay = true,
                silent = true
            })

            -- 3. Verify target bar appeared (up to 5 x 50ms)
            local locked = false
            for _ = 1, 5 do
                nhk.sleep(50)
                if not isRunning then break end

                local target = nhk.getTargetStream({ client = CLIENT })
                if target.hasTarget and (target.isMonster or not target.isDead) then
                    locked = true
                    break
                end
            end

            -- 4. Combat state machine: Engaging -> Fighting -> Target Dead
            if locked then
                nhk.pressKey({ client = CLIENT, key = ATTACK_KEY, silent = true })

                local initialTarget = nhk.getTargetStream({ client = CLIENT })
                local lastHp = initialTarget.hp or 100
                local isActivelyFighting = false
                local retryAttackTimer = 0

                while isRunning do
                    nhk.sleep(50)
                    if not isRunning then break end

                    local target = nhk.getTargetStream({ client = CLIENT })
                    if target.hasTarget then
                        local currentHp = target.hp or 100
                        if currentHp < lastHp then
                            isActivelyFighting = true
                            lastHp = currentHp
                        end

                        if isActivelyFighting then
                            nhk.setStatusOverlay({
                                client = CLIENT,
                                name = "auto_hunt",
                                status = "Fighting",
                                detail = string.format("HP: %.0f%%", currentHp),
                                type = "action"
                            })
                        else
                            retryAttackTimer = retryAttackTimer + 1
                            if retryAttackTimer >= 30 then
                                nhk.pressKey({ client = CLIENT, key = ATTACK_KEY, silent = true })
                                retryAttackTimer = 0
                            end
                            nhk.setStatusOverlay({
                                client = CLIENT,
                                name = "auto_hunt",
                                status = "Engaging",
                                detail = string.format("HP: %.0f%% (Approaching)", currentHp),
                                type = "action"
                            })
                        end
                    else
                        break -- Target bar vanished -> mob dead
                    end
                end
                nhk.sleep(50)
            else
                nhk.sleep(50)
            end
        else
            nhk.sleep(80)
        end
    end
end)
```

### Example C: External Python Script Triggering NHK4 via HTTP Webhook
```python
import requests

# Trigger nhk.onWebhook({ route = "emergency-buff", ... })
resp = requests.post(
    "http://127.0.0.1:3088/api/webhook/emergency-buff",
    json={"client": 2, "keys": ["F1", "F2", "F3"]}
)
print(resp.json())
```

### Example D: Web Extraction + HTTP + Local JSON Persistence
```lua
nhk.onHotkey("home", function()
    -- 1. Extract data from a normal website open in Client 2
    nhk.webNavigate({ client = 2, url = "https://www.wikipedia.org/" })
    if nhk.webWaitFor({ client = 2, selector = "#searchInput", timeoutMs = 5000 }) then
        nhk.webType({ client = 2, selector = "#searchInput", text = "Rust (programming language)" })
        nhk.webClick({ client = 2, selector = "button[type=\"submit\"]" })
        if nhk.webWaitFor({ client = 2, selector = "#firstHeading", timeoutMs = 8000 }) then
            local title = nhk.webQuery({ client = 2, selector = "#firstHeading" })
            nhk.writeJson({
                path = "C:/Users/%USERNAME%/Documents/NHK4/web_result.json",
                data = { heading = title, url = nhk.webInfo({ client = 2 }).url }
            })
        end
    end
end)
```

### Example E: WASM Next-Scan + Read Loop
```lua
local hits = nil
nhk.onHotkey("insert", function()
    -- Press once when HP=100, again when HP=80 (update targetVal each time)
    local targetVal = 100
    hits = nhk.wasmScan({ client = 1, value = targetVal, type = "i32", offsets = hits })
    nhk.log("info", "WASM scan hits=" .. tostring(hits and #hits or 0))
end)
```

---

## 2. Copy-Paste Prompt Templates for External AIs

### Prompt 1: Ask an AI to Write a New NHK4 Luau Script
> You are an expert Luau developer for **NHK4 (NodeHotKeyV4)**.
> Read the attached NHK4 Wiki (`AI_CONTEXT_BUNDLE.md` or the SDK reference) carefully.
>
> **Strict Rules:**
> 1. Only use real functions from the `nhk.*` SDK documented in the Wiki. Do not invent APIs.
> 2. Write **event-driven** code using `nhk.onHotkey`, `nhk.onEvent`, `nhk.onWebhook`, `nhk.onOverlayInput`, and `nhk.setTimer` instead of infinite `while true` polling loops.
> 3. Always use `nhk.wait({ ms = ... })` if any loop is needed, and use `silent = true` in high-frequency timers.
> 4. Keep all `nhk.log` messages in English ASCII.
> 5. Expose user-configurable values (thresholds, toggles) with `nhk.registerOverlayInput` instead of magic hardcoded numbers where appropriate.
> 6. Use `"home"`, `"insert"`, or `"delete"` for bot start/stop toggle hotkeys — do **not** bind `F1`–`F12` as bot toggle hotkeys because they are used by in-game systems/skill bars.
>
> **My Task:**
> [Describe what you want the script to do here, e.g., Client 1 attacks, Client 2 heals when HP < 75% and buffs every 5 minutes.]

### Prompt 2: Ask an AI to Connect an External App (Python/Node.js/Stream Deck) to NHK4
> Using the attached NHK4 REST & Webhook Wiki (`03-REST-API-Webhooks-And-Events.md`), write:
> 1. An external [Python / Node.js] client connecting to `http://127.0.0.1:3088` (`/api/webhook/...`, `/api/trigger/...`, `/api/overlay/input`, or `/api/signals/stream`).
> 2. The matching NHK4 Luau script using `nhk.onWebhook` / `nhk.onEvent` to execute the in-game actions on the specified client slot.

