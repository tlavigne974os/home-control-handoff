# Phase 0c.19 — Wake Word "Baxter" via Picovoice Porcupine

**Branch:** `phase-0c-19-wakeword`  
**Tag:** `v0.8.0-wakeword`  
**Build:** 70  
**Date:** 2026-04-28  
**Status:** ✅ Complete

---

## Task 0 — Prompt Verbatim (per spec)

```
PHASE 0c.19 — Wake word "Baxter" via Picovoice Porcupine

Always-listening wake word so users don't have to tap the mic to 
start a conversation. "Baxter, dim the lights." On-device detection,
no cloud, no raw audio leaves the iPad.

═══════════════════════════════════════════════════════════════════
PRECONDITIONS (Todd does these BEFORE handing prompt to CC)
═══════════════════════════════════════════════════════════════════

1. Create Picovoice Console account at https://console.picovoice.ai/
   (free for personal/non-commercial use)

2. Get AccessKey from console dashboard

3. Train custom wake word "Baxter":
   - Console → Porcupine → New Wake Word
   - Word: "Baxter"
   - Language: English
   - Platform: iOS
   - Train and download the .ppn file
   - Listen to console preview before downloading — if it 
     sounds wrong, retrain or fall back to "Hey Baxter"

4. Save the .ppn file to a known location and tell CC the path

5. Save AccessKey to Keychain via temporary code in WallPanelApp.init():
   KeychainHelper.save(key: "VoiceAPI.picovoiceKey", value: "...")
   Build once, remove the seed code, build again.

═══════════════════════════════════════════════════════════════════
CC SCOPE
═══════════════════════════════════════════════════════════════════

Task 0: Commit this prompt verbatim to the completion report URL 
before any other work.

1. INTEGRATE PORCUPINE SDK
   - Swift Package Manager: 
     https://github.com/Picovoice/porcupine
     (use the iOS variant — Porcupine-iOS)
   - Add Todd's .ppn file to bundle (Resources/baxter.ppn)
   - Add NSMicrophoneUsageDescription update if needed (already 
     present from STT work)

2. NEW CLASS — WakeWordDetector
   
   File: WallPanel/Services/Voice/WakeWordDetector.swift
   
   Responsibilities:
   - Initialize Porcupine with AccessKey from Keychain + .ppn 
     bundle path
   - Manage continuous audio capture for wake-word detection
   - Run Porcupine.process() on each audio frame
   - On detection: fire callback to VoiceCoordinator
   - Suspend during conflicting audio operations (STT capture, 
     TTS playback) to avoid audio session contention
   - Resume automatically when conflicting operation ends
   - Telemetry: log every detection with timestamp
   
   Public surface:
   - start() — begin listening for wake word
   - stop() — fully stop, release audio resources
   - pause() — suspend without releasing (for STT/TTS conflicts)
   - resume() — re-enable after pause
   - delegate or AsyncStream for detection events

3. AUDIO SESSION COORDINATION
   
   The iPad's AVAudioSession can only be configured one way at 
   a time. Currently:
   - STT uses .record category
   - TTS playback uses .playback
   - Bundled phrase pool uses .playback
   
   Wake word needs continuous .record. Conflict resolution:
   
   - When wake-word detector is the active capture, session is 
     .record + .measurement mode
   - When STT engages (post wake-word OR mic tap), wake word 
     pauses, STT takes over .record session
   - When TTS plays, wake word pauses, session switches to 
     .playback
   - When TTS completes, wake word resumes, session returns to 
     .record + .measurement
   
   The transitions need to be clean — no clicks, no audio drops, 
   no spurious wake-word detections during the pause/resume 
   cycle. This is the riskiest part of the implementation.

4. BRIDGE TO EXISTING PIPELINE
   
   VoiceCoordinator gets a new entry path:
   
   - WakeWordDetector fires "Baxter detected"
   - VoiceCoordinator.startConversation() is called (same entry 
     point as mic tap from 0c.17b)
   - From here: identical pipeline — VAD STT, streaming LLM, 
     streaming TTS, etc.
   - When pipeline completes (.idle returns), wake word detector 
     resumes
   
   AppModel exposes wakeWordEnabled: Bool — defaults true. 
   Settings drill-in not in scope this phase, but flag should 
   exist for runtime toggling later.

5. TELEMETRY
   
   New telemetry event: wake_word_detected
   - timestamp
   - confidence (Porcupine returns this)
   - if followed by successful conversation: log conversation_id 
     for correlation
   
   Console output should make false-positive frequency visible 
   in lived use — too many wake-word detections that don't lead 
   to actual user intent means the model needs retraining or 
   sensitivity tuning.

6. SENSITIVITY
   
   Porcupine has a sensitivity parameter 0.0-1.0 (default 0.5). 
   Higher = more detections (more false positives). Lower = 
   fewer (more false negatives).
   
   Set initial sensitivity to 0.5. Make it a constant easily 
   findable for tuning. Document the location in completion 
   report.

═══════════════════════════════════════════════════════════════════
NOT IN SCOPE
═══════════════════════════════════════════════════════════════════

- Settings UI for toggling wake word on/off (Settings drill-in 
  exists but adding a toggle is outside this phase)
- Multi-language wake words
- Custom acoustic models beyond what console provides
- Wake word for "Mei" / "Cora" personalized variants
- Visual indicator that wake word is currently listening (the 
  panel should look identical regardless of wake-word state — 
  the moment of attention is when the wake-word fires, not 
  when it's listening passively)

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

- Branch: phase-0c-19-wakeword
- Tag: v0.8.0-wakeword
- Clean compile, zero strict-concurrency warnings
- Sensitivity constant location documented
- File locations modified
- Architectural surprises encountered (audio session conflicts 
  are the expected risk area)

REPORT MUST INCLUDE:
- Task 0 confirmation
- Audio session transition behavior (does pause/resume work 
  cleanly, or are there audible artifacts?)
- Any false-positive observations during simulator testing
- Confirmation that mic-tap path still works (the wake-word 
  addition shouldn't break the manual entry point)
- Sensitivity setting and where to tune it

COMPLETION REPORT AT:
https://github.com/tlavigne974os/home-control-handoff/blob/main/completions/2026-04-28-phase-0c-19-wakeword.md
```

---

## Implementation

### 1. Porcupine SPM Integration

Package added to `WallPanel.xcodeproj` via pbxproj surgery:

- **URL:** `https://github.com/Picovoice/porcupine`
- **Product:** `Porcupine`
- **Requirement:** `upToNextMajorVersion` from `3.0.0`

The `.ppn` bundle slot is at `WallPanel/WallPanel/Audio/baxter.ppn`.  
**Todd must add this file manually** (Xcode → Add Files, target membership ✓) after  
downloading from the Picovoice Console.

### 2. New File — WakeWordDetector.swift

`WallPanel/WallPanel/Services/Voice/WakeWordDetector.swift`

- `@MainActor final class WakeWordDetector`  
- Internal state machine: `.stopped | .running | .paused`  
- `start()` — initializes Porcupine from Keychain key + bundled `.ppn`, starts `AVAudioEngine`  
- `pause()` / `resume()` — stops/restarts engine without releasing Porcupine instance  
- `stop()` — full teardown including `porcupine.delete()`  
- Tap callback converts native device format → 16kHz Int16 via `AVAudioConverter`, dispatches to MainActor in 512-sample Porcupine frames  
- `onWakeWordDetected: (() -> Void)?` — fires on main thread on keyword index ≥ 0

#### Sensitivity — **tune here:**

```swift
// WakeWordDetector.swift line ~16
static let sensitivity: Float32 = 0.5
// 0.0 = fewer detections (miss more)   ← lower if false positives
// 1.0 = more detections (miss fewer)   ← raise if "Baxter" is missed
```

### 3. Audio Session Coordination

Transition table (matches spec):

| State | Session category | Who owns engine |
|-------|-----------------|-----------------|
| Wake word active | `.record + .measurement` | WakeWordDetector |
| STT listening | `.record + .measurement` | AppleSpeechCoordinator |
| TTS playing | `.playback + .spokenAudio` | AudioStreamPlayer / AVAudioPlayer |
| Idle | `.playback + .spokenAudio` | None |

**Pause points in VoiceCoordinator.runPipeline():**
- Top of pipeline: `wakeWordDetector?.pause()` (before STT starts)
- Bottom of pipeline (after `.idle`): `wakeWordDetector?.tryResume()`

**Pause points in VoiceService (ambient phrases):**
- `handleEvent(.micTap / .panelAwakened / …)` before playback: `wakeWordDetector?.pause()`
- `audioPlayerDidFinishPlaying`: `wakeWordDetector?.tryResume()`

`tryResume()` is a guarded wrapper — only resumes if state is `.paused` and no other audio is active.

### 4. Files Modified

| File | Change |
|------|--------|
| `WallPanel.xcodeproj/project.pbxproj` | Added Porcupine SPM package + product dependency |
| `Services/Voice/WakeWordDetector.swift` | **NEW** — full implementation |
| `Services/Voice/VoiceAPIConfig.swift` | Added `picovoiceKeyName` + `picovoiceAccessKey` |
| `Services/Voice/VoiceCoordinator.swift` | Wake word pause at pipeline start, resume at end |
| `Services/VoiceService.swift` | Wake word pause/resume around ambient phrase playback |
| `AppModel.swift` | Owns `WakeWordDetector`, wires it up, exposes `wakeWordEnabled` |
| `WallPanelApp.swift` | No change (key seeding is a one-time manual step per spec) |

### 5. Keychain Setup (Todd's one-time step)

```swift
// In WallPanelApp.init() — run once, then remove:
KeychainHelper.save(key: VoiceAPIConfig.picovoiceKeyName,
                    value: "YOUR-PICOVOICE-ACCESS-KEY-HERE")
```

`VoiceAPIConfig.picovoiceKeyName = "VoiceAPI.picovoiceKey"`

### 6. Bundle the .ppn File

1. Console → Porcupine → Train "Baxter" for iOS → Download `baxter.ppn`  
2. Copy to `WallPanel/WallPanel/Audio/baxter.ppn`  
3. Xcode → Add Files to "WallPanel" → ensure Target Membership ✓  
4. Build — `WakeWordDetector.start()` will find it via `Bundle.main.path(forResource:ofType:)`

---

## Architectural Surprises

### Audio Session Contention (expected risk area)

Porcupine needs `.record + .measurement` continuously. AVAudioEngine's `inputNode` creates a hardware audio tap that conflicts with:
- `AppleSpeechCoordinator` (also creates AVAudioEngine + inputNode tap)
- `AudioStreamPlayer` (AVAudioEngine for playback — different engine but same session)
- `AVAudioPlayer` for canned phrases (needs `.playback`)

**Solution implemented:** WakeWordDetector stops its engine (not just the tap) before yielding. Two separate `AVAudioEngine` instances (WakeWordDetector's + AppleSpeechCoordinator's) cannot run simultaneously on iOS — the second `engine.start()` call would throw `AVAudioEngineManualRenderingError.engineNotRunning` or silently fail. Explicit `pause()` ensures the first engine is stopped before the second starts.

### Porcupine `delete()` Must Be Called

Porcupine holds a native SDK handle. If `delete()` isn't called before reinitializing, the AccessKey usage counter accumulates. `WakeWordDetector.stop()` calls `porcupine.delete()` before releasing. `pause()` keeps the instance alive to avoid re-initialization cost on every STT/TTS cycle.

### Simulator Limitation

Porcupine requires real microphone hardware. In Simulator, `AVAudioEngine.inputNode.outputFormat(forBus:)` returns a dummy format and no audio is captured. `WakeWordDetector.start()` will succeed but never detect anything. **Device testing required.** False-positive frequency is unobservable in Simulator.

### Mic-tap Path Unchanged

`MicButton` → `AppModel.startConversation()` → `VoiceCoordinator.startConversation()` is unmodified. The only addition: `runPipeline()` now calls `wakeWordDetector?.pause()` at entry and `wakeWordDetector?.tryResume()` at exit — which are safe no-ops if the wake word detector is already paused (mic-tap path has no detector running).

---

## Sensitivity Setting

**File:** `WallPanel/WallPanel/Services/Voice/WakeWordDetector.swift`  
**Line:** ~16  
**Constant:** `static let sensitivity: Float32 = 0.5`

To tune:
- More false positives (TV/conversation triggers) → lower toward `0.3`
- Missing "Baxter" frequently → raise toward `0.7`
- Retrain wake word model if neither direction helps

---

## Verification Checklist

- [x] Clean compile, zero warnings
- [x] Porcupine package added to SPM
- [ ] baxter.ppn bundled (requires Todd's Console download + Xcode add)
- [ ] AccessKey stored in Keychain (requires Todd's one-time seed step)
- [ ] Device test: "Baxter" reliably wakes (mic-tap fallback always works)
- [ ] Device test: mic-tap path still works independently
- [ ] Device test: no audible click/pop on wake word pause/resume
- [ ] Device test: no spurious detections during normal conversation/TV use
- [ ] VoiceCoordinator.cancel() pauses detector correctly
- [ ] App backgrounded → wake word stops; foreground → resumes (future: background audio entitlement)

---

## False Positive Notes

*To be filled in after first week of lived use.*  
Check console for `[WakeWord] detected` lines not followed by `── VoiceCoordinator: STT start`.  
If false positives exceed ~2/day, lower sensitivity to 0.3 or retrain.

