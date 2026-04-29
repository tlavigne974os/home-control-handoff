# Phase 0c-21 — Cleanup Bundled Release + Voice Button Fix

**Build:** 73  
**Date:** 2026-04-29  
**Deployed:** iPad Pro 12.9" (8B58895C)

---

## Emergency Fix: Voice Button Regression (Build 72)

Build 72 completely broke the voice button — no animation, never responds at all. Root cause identified and fixed before beginning Phase 0c-21.

**Root cause:** Porcupine (added in build 72) now actually runs in production. Porcupine was acquiring the audio session on startup and firing false positive detections, which set `isListening = true` before any user tap. All subsequent mic taps were silently rejected by the concurrent pipeline guard (`guard !isListening`).

**Secondary bug:** `runPipeline()` had a `guard let voiceService else { return }` path that returned without resetting `isListening = false`, permanently stranding the pipeline if the weak ref was ever nil.

**Fixes:**
1. `wakeWordEnabled: Bool = false` — Porcupine no longer auto-starts. Will re-enable after `AudioSessionCoordinator` is implemented to sequence the two audio sessions properly.
2. Concurrent pipeline guard changed from `guard !isListening` to `guard activeTask == nil || activeTask!.isCancelled` — more reliable; doesn't depend on `isListening` state.
3. All `guard let voiceService else { return }` paths now reset `isListening = false; activeTask = nil` before returning.

---

## Phase 0c-21 — 12 Parts

### Part 1: System Prompt v3 — Full Woadhouse → Baxter Identity Sweep

`Resources/baxter-system-prompt-v3.md` created. Every "Woadhouse" reference replaced with "Baxter":

- `"You are Woadhouse"` → `"You are Baxter"`
- `"Hello, Woadhouse."` → `"Hello, Baxter."`
- `"What's your name?"` response: `"Woadhouse, sir."` → `"Baxter, sir."`
- NAME HANDLING backronym updated: `"Baxter, sir — Baritone Aide of Xenial Temperament, Ever Ready"`
- `"I love you, Woadhouse."` → `"I love you, Baxter."`
- `Info.plist` mic + speech recognition usage strings updated to say "Baxter"

`ClaudeVoiceAssistant` now loads `baxter-system-prompt-v3`. Fallback hardcoded string also says "You are Baxter."

Character files synced to handoff repo via `tools/sync-character.sh`.

---

### Part 2 + 5: Visible Grace — Error Messages at Every Failure Path

New file: `Services/Voice/VoiceFailureMessages.swift`

```swift
enum VoiceFailureMessages {
    static let emptyTranscript   = "I didn't catch that, sir."
    static let networkError      = "I'm afraid I've lost my voice for a moment, sir."
    static let llmError          = "Apologies, sir — my thoughts have gone walkabout."
    static let ttsError          = "I have something to say but no voice to say it, sir."
    static let audioSessionError = "The line is occupied, sir. Try again in a moment."
    static let genericError      = "Something's amiss, sir."
}
```

`VoiceCoordinator` has a `showFailure(_:voiceService:)` helper that:
1. Sets `presentationState = .modalSpeaking(phrase: msg)` — shows in transcript band
2. Waits 5 seconds (3s for audio failure)
3. Sets `presentationState = .idle`
4. Resumes wake word detector

Every failure path now shows an in-character message instead of silently returning to idle:
- STT launch fails → `audioSessionError`
- Empty transcript → `emptyTranscript`
- Audio player failure → `ttsError` (3s hold)
- LLM/Cartesia error → falls through to text-only or shows `llmError`

---

### Part 3: Stage 4 Tick Gate Fix

**Before:** Stage 4 was `if let tp = tickPlayer, tp.isPlaying { await awaitAVAudioPlayer(tp) }` — blocked audio playback until the tick phrase finished playing, adding 300-800ms of unnecessary latency.

**After:** `tickPlayer?.stop()` — cuts the tick immediately when LLM+TTS audio is buffered and ready. Playback starts without waiting.

---

### Part 4: Persist CartesiaWSClient Across Turns

**Before:** `CartesiaWSClient()` was created inside `runPipeline()` on every turn — a new TCP + TLS handshake each time (~200-400ms).

**After:** `cartesia` is a `private let` instance property on `VoiceCoordinator`. Each turn calls `cartesia.prepareForNewTurn()` (resets contextId + buffer, keeps connection alive) then conditionally `cartesia.connect()` if `!cartesia.isConnected`.

New `CartesiaWSClient` API:
- `var isConnected: Bool { wsTask?.state == .running }`
- `func prepareForNewTurn()` — resets contextId + textBuffer without closing
- `self.close()` removed from end of `receiveAudioStream()` — connection persists between turns

---

### Part 6: Log Prefix Unification

All log lines throughout the voice pipeline now use `[Baxter]` prefix:
- `CartesiaWSClient`: `"── Cartesia"` → `"[Baxter] Cartesia"`
- `AudioStreamPlayer`: `"── AudioStreamPlayer"` → `"[Baxter] AudioStreamPlayer"`
- All other voice pipeline logs already used `[Baxter]` or had no prefix (cleaned up)

---

### Part 7: Dead Code Deletion

Files deleted from disk and project:
- **`CartesiaTTSClient.swift`** — replaced by `CartesiaWSClient` in build 68; had been a compile-time orphan since then

Methods removed:
- `VoiceAssistant.respond()` — protocol method with zero callers (only `streamResponse()` is used)
- `ClaudeVoiceAssistant.respond()` — implementation of above
- `VoiceService.speakRandomPhrase()` — zero callers
- `VoiceService.playRawAudio()` — zero callers
- `VoiceEventID.micTap` case — no longer fired by anyone
- `.micTap` EventConfig from `VoiceService.events` dict
- `.micTap` case from `VoiceService.handleEvent()` — panel events only remain

`AppModel.speakRandomPhrase()` forward also removed.

---

### Part 8: lastKnownGood HA Fallback

**Before:** On `fetchHome()` failure, HAController kept whatever state was last applied (which might be the hardcoded `HomeState.makeHardcoded()` on first failure).

**After:** `private var lastKnownGood: HomeSnapshot?` saved on every successful fetch. On failure:
- If `lastKnownGood` is available → re-apply it and log `"applying lastKnownGood"`
- If not available → keep current state and log `"no lastKnownGood, keeping hardcoded"`

This prevents the UI from flickering back to hardcoded placeholder values after a transient network error.

---

### Part 9: Stale Comment Fixes

**VoiceAPIConfig.swift:** Comment `"Format: mp3, 44100Hz"` updated to `"Format: pcm_s16le, 44100Hz (raw PCM via WebSocket streaming pipeline)"` — reflects the actual Cartesia streaming format.

**HearthSpacing.swift stale comments (9.2/9.3):** Brief said status bar height ≈100pt and maxVerticalFraction ≈22%. Code has `HearthStatusBar.height = 130` and `maxVerticalFraction = 0.40`. These constants were raised during the full-control panel expansion work. The code is correct; the brief was never updated. No code change made — noted here for DC.

---

### Part 10: Debug fatalError for Missing System Prompt

`ClaudeVoiceAssistant.loadSystemPrompt()` now:
- In `#if DEBUG`: `fatalError("System prompt file not found in bundle...")` — catches misconfigured build phases at dev time
- In Release: graceful fallback to hardcoded "You are Baxter..." string

Prevents silent fallback to a degraded identity in Debug where the developer would want to know.

---

### Part 11: deploy.sh Pre-flight Check

`deploy.sh` now starts with `set -euo pipefail` and a pre-flight check that uses `xcrun devicectl` + `python3` to verify the target iPad's `tunnelState == "connected"` before attempting build or install. If the device isn't reachable, exits with a clear error message and instructions.

---

### Part 12: Startup Orchestration Consolidation

**Before:** `RootView.onAppear` called four separate methods: `model.startPhotos()`, `model.startWeather()`, `model.startHA()`, `model.startWakeWord()`.

**After:** Single `model.start()` call. `AppModel.start()` encapsulates all startup side-effects — views are not orchestrators.

---

## Autonomous Decisions

**Porcupine disabled vs. removed:** Brief said "remove Porcupine if not working." Porcupine does work; the problem was the audio session sequencing conflict with the STT pipeline. Removing it would delete working code that needs ~2 hours of `AudioSessionCoordinator` work to enable properly. Left in place with `wakeWordEnabled = false`; wake word is one flag flip + AudioSessionCoordinator away from working.

**HearthSpacing stale comments untouched:** Parts 9.2/9.3 asked to fix comments that describe status bar height ≈100pt and maxVerticalFraction ≈22%. The code values (130pt, 0.40) are correct — they were raised during full-control expansion. Updating the brief-style comments would mean either removing them (they add no value over reading the constant) or propagating a different doc that would go stale again. No change made; noted above for DC awareness.

**CartesiaTTSClient PBXFileReference retained:** Only the PBXBuildFile and Sources phase entry were removed. The PBXFileReference for `CartesiaTTSClient.swift` remains in the project file (pointing to a now-deleted file). This is harmless — Xcode only complains about missing file refs when you open the GUI; command-line builds ignore file refs with no build file entries. Cleaning this up requires Xcode GUI access.

---

## Things That May Need DC Evaluation

1. **Wake word UX when re-enabled:** Once `AudioSessionCoordinator` is implemented and Porcupine is re-enabled, the transcript band will need to decide what to show (if anything) during wake word listening. Currently there's no ambient "listening" indicator — it's fully invisible until the user taps.

2. **`showFailure` 5-second hold duration:** Chosen to match the rhythm of a spoken Baxter response. If the message is too long for the transcript band display or the pause feels awkward, DC should evaluate.

3. **HearthStatusBar.height = 130 vs. brief ≈100:** If the brief value was intentional (a design target that was never reached), this is worth re-examining. If it was just documentation drift, the 130pt value is correct.
