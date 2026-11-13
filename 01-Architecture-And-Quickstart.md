# 01. Architecture & Quickstart

## 1. What is NHK4?
**NHK4 (NodeHotKeyV4)** is a native **Rust + Tokio** automation and multi-client orchestration engine built for browser/desktop games (such as Flyff Universe). It combines:
- **Chrome DevTools Protocol (CDP)** over WebSocket for background multi-client input & screenshot capture.
- **OS-Level Input Synthesis & Global Hotkey Listener** (`rdev` / Win32 APIs).
- **Sandboxed Luau VM (`mlua`)** for writing ultra-fast, event-driven automation scripts (`nhk.*` / `bot.*`).
- **Axum HTTP & SSE Server** on `http://127.0.0.1:3088` for UI, Launcher, Overlay, PiP, and external integrations.

---

## 2. Dual-Mode Profiles (Visual Node vs. Luau Code)
NHK4 supports two profile modes that can run **simultaneously** and communicate via `nhk.emitEvent` / `nhk.onEvent`:
1. **Visual Node Mode (`node`):** Blueprint-style graph of modular nodes.
2. **Luau Code Mode (`code`):** Native Luau script compiled to bytecode on engine start. Each active Code profile runs in its own isolated `LuauRuntime` VM (`LuauRuntime::new_for_profile`), so reloading one profile does not stop other running profiles.

---

## 3. Dynamic Client Slots (Fixed Slot IDs)
Each game window is bound to a **permanent numeric Slot ID** (`1`, `2`, `3`, ...).
- Slots are **never auto-reindexed** when another slot is removed.
- Special target strings accepted by input/client APIs:
  - `1` or `"1"`: Target Client Slot 1
  - `"1,2"` or `{1, 2}`: Target multiple client slots
  - `"all"`: All active/configured client slots
  - `"global"`, `"0"`, or `"os"`: Host OS cursor / global desktop target (for `mouseClick`, `mouseMove`, `isClientOnline`)

---

## 4. Luau Sandbox & Execution Rules (CRITICAL FOR AI)
When generating Luau scripts for NHK4, you **must** follow these runtime rules:

1. **Use `nhk.*` (or `bot.*` alias):**
   - Both `nhk` and `bot` point to the same read-only SDK table (`bot.set_readonly(true)`).
   - Standard Luau libraries (`math`, `string`, `table`, `bit32`, `os.clock`, `os.time`, etc.) are sandboxed and read-only. File I/O (`io`), `os.execute`, and arbitrary OS process spawning are **blocked**.
2. **Event-Driven First — Never Block with Busy Loops:**
   - Do **not** write `while true do end` without yielding.
   - The VM enforces an instruction budget (`DEFAULT_MAX_INSTRUCTIONS_WITHOUT_WAIT = 200,000` instructions without yielding). Exceeding it triggers `ERR_INSTRUCTION_LIMIT`.
   - Always prefer **event callbacks** (`nhk.onHotkey`, `nhk.onEvent`, `nhk.onWebhook`, `nhk.onOverlayInput`) and **non-blocking timers** (`nhk.setTimer`).
   - If you must loop, always yield with `nhk.wait({ ms = ... })` (which resets the instruction counter and cancels immediately on Engine Stop / Emergency Suspend with `ERR_ENGINE_STOPPED`).
3. **Dual Syntax Support:**
   - Almost every `nhk.*` function supports **both**:
     - **Table Syntax (Recommended):** `nhk.pressKey({ client = 1, key = "F1", hold = 50 })`
     - **Positional Syntax:** `nhk.pressKey(1, "F1", 50)`
4. **Silent Flag (`silent = true`):**
   - Many table-syntax calls accept `silent = true` to suppress action log spam in high-frequency timers/loops.
5. **Pro License Guard:**
   - Vision & TTS functions (`scanParty`, `partyHeal`, `partyBuffNext`, `selectPartySlot`, `takeScreenshot`, `takeDesktopScreenshot`, `scanMonsters`, `targetMonster`, `scanMonstersStream`, `getTarget`, `getTargetStream`, `speakTts`) check for a Pro license (`can_use_pro_features()`). On Trial/Lite tiers they log a warning and return empty/false safely.

