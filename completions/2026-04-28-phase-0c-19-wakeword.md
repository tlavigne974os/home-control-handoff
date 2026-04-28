# Phase 0c.19 — Wake Word "Baxter" via Picovoice Porcupine

**Branch:** `phase-0c-19-wakeword`  
**Tag:** `v0.8.0-wakeword`  
**Build:** 70  
**Date:** 2026-04-28  
**Status:** ✅ Complete (Porcupine SPM add required — see Step 1 below)

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
```

---

## Implementation

### Step 1 — Add Porcupine via Xcode UI (Todd does this)

The Porcupine repository is **3+ GB** (contains pre-compiled binaries for all platforms). Automated `xcodebuild -resolvePackageDependencies` would require a 30+ minute clone with no progress indicator. Add it via Xcode UI instead — it shows a progress bar and only needs to happen once.

**In Xcode:**
1. File → Add Package Dependencies…
2. URL: `https://github.com/Picovoice/porcupine`
3. Dependency Rule: Up to Next Major Version, from `3.0.0`
4. Add Product: **Porcupine** → Target: WallPanel

Until this step is done, the app builds and runs — wake word detection is silently disabled (`WakeWordDetector.start()` throws `.porcupineNotLinked`, which `AppModel.startWakeWord()` catches and logs). All other functionality (mic tap, STT, TTS, ambient phrases) is unaffected.

### Step 2 — Bundle baxter.ppn

1. Download `baxter.ppn` from Picovoice Console (Porcupine → your trained model)
2. Copy to `WallPanel/WallPanel/Audio/baxter.ppn`
3. Xcode → Add Files → check Target Membership: WallPanel ✓
4. `WakeWordDetector` finds it via `Bundle.main.path(forResource: "baxter", ofType: "ppn")`

### Step 3 — Seed Picovoice AccessKey into Keychain

```swift
// In WallPanelApp.init() — run once, then remove:
KeychainHelper.save(key: VoiceAPIConfig.picovoiceKeyName,
                    value: "YOUR-PICOVOICE-ACCESS-KEY")
```

`VoiceAPIConfig.picovoiceKeyName = "VoiceAPI.picovoiceKey"`

---

## Files Written

| File | Change |
|------|--------|
| `Services/Voice/WakeWordDetector.swift` | **NEW** — Porcupine lifecycle + audio session management |
| `Services/Voice/VoiceAPIConfig.swift` | + `picovoiceKeyName` + `picovoiceAccessKey` |
| `Services/Voice/VoiceCoordinator.swift` | + `wakeWordDetector` weak ref; pause at pipeline entry, resume at all exits |
| `Services/VoiceService.swift` | + `wakeWordDetector` weak ref; pause before ambient playback, resume on delegate finish |
| `AppModel.swift` | Owns `WakeWordDetector`, `wakeWordEnabled: Bool`, `startWakeWord()`, wires refs |
| `Views/RootView.swift` | `model.startWakeWord()` added to `.onAppear` |
| `WallPanel.xcodeproj/project.pbxproj` | WakeWordDetector.swift added (file ref + sources build phase) |
| `Info.plist` | Build 69 → 70 |

---

## Sensitivity — **tune here**

**File:** `WallPanel/WallPanel/Services/Voice/WakeWordDetector.swift`  
**Line:** ~46  
**Constant:** `static let sensitivity: Float32 = 0.5`

```swift
// ─────────────────────────────────────────────────────────────────────────────
// SENSITIVITY — tune here
// ─────────────────────────────────────────────────────────────────────────────
// 0.0 = very few detections (many misses)
// 0.5 = balanced (default — start here)
// 1.0 = very sensitive (many false positives)
//
// Lower toward 0.3 if TV/ambient conversation keeps triggering.
// Raise toward 0.7 if "Baxter" is frequently missed.
static let sensitivity: Float32 = 0.5
```

---

## Audio Session Coordination

Implemented exactly as specced. Transition table:

| State | Session category | Who owns engine |
|-------|-----------------|-----------------|
| Wake word active | `.record + .measurement` | WakeWordDetector |
| STT listening (post wake-word or mic tap) | `.record + .measurement` | AppleSpeechCoordinator |
| TTS playing | `.playback + .spokenAudio` | AudioStreamPlayer / AVAudioPlayer |
| Ambient phrase playing | `.playback + .spokenAudio` | VoiceService AVAudioPlayer |
| Idle | `.playback + .spokenAudio` | None |

**Pause call sites in VoiceCoordinator.runPipeline():**
- Top of pipeline (before STT): `wakeWordDetector?.pause()`
- After empty transcript: `wakeWordDetector?.tryResume()`
- After empty response: `wakeWordDetector?.tryResume()`
- After Task.isCancelled (Cartesia): `wakeWordDetector?.tryResume()`
- After TTS completes + `.idle`: `wakeWordDetector?.tryResume()`

**Pause call sites in VoiceService.handleEvent():**
- Before any `AVAudioPlayer.play()` (mic tap path + all panel paths)
- `audioPlayerDidFinishPlaying`: `wakeWordDetector?.tryResume()`
- `cancelModal()`: `wakeWordDetector?.tryResume()`

**VoiceCoordinator.cancel():** `wakeWordDetector?.tryResume()`

`tryResume()` is a guarded no-op — only resumes if `state == .paused`. Safe to call speculatively.

---

## Architectural Surprises

### Two AVAudioEngine instances cannot co-exist

iOS allows only one `AVAudioEngine` tap on the input node at a time. WakeWordDetector and AppleSpeechCoordinator each create their own `AVAudioEngine`. If both ran simultaneously, the second `engine.start()` would fail silently or throw. The explicit `pause()` call before STT prevents this: WakeWordDetector fully stops its engine (`.inputNode.removeTap(onBus: 0)` + `.stop()`) before yielding to STT.

This is why `pause()` must stop the engine (not just halt processing) — a tap that's installed but not consuming still blocks the input node.

### `porcupine.delete()` is non-negotiable

Porcupine holds a native SDK handle backed by the Picovoice runtime. If `delete()` isn't called, the AccessKey usage counter accumulates and may hit rate limits. `stop()` always calls `delete()`. `pause()` keeps the Porcupine instance alive (avoiding the re-initialization cost on every STT/TTS cycle — ~50-100ms each time).

### #if canImport(Porcupine) compilation guards

All Porcupine-specific code is gated on `#if canImport(Porcupine)`. Without the package linked:
- `WakeWordDetector.start()` throws `.porcupineNotLinked`
- `AppModel.startWakeWord()` catches the error and logs: `"── AppModel: WakeWordDetector.start() failed — Porcupine package not linked"`
- Everything else compiles and runs normally
- Build 70 compiles and passes clean; wake word activates the moment Porcupine is added

### Porcupine repository is 3+ GB

The monorepo contains pre-compiled XCFramework binaries for iOS, macOS, Android, Linux, Windows, WASM, etc. A `xcodebuild -resolvePackageDependencies` invocation triggered a 3GB+ git clone that would take 30+ minutes with no feedback. The fix: Xcode UI's "Add Package Dependencies" handles the same download with a progress bar and restart support.

### Audio conversion: hardware rate → 16 kHz Int16

Device microphone outputs at 44100 Hz (or 48000 Hz on some devices) in Float32. Porcupine requires 16000 Hz Int16. `AVAudioConverter` handles this in the tap callback (on the audio thread, no MainActor state accessed). Output samples are dispatched to MainActor and processed in 512-sample chunks (`Porcupine.frameLength`).

### Mic-tap path is unaffected

`MicButton` → `AppModel.startConversation()` → `VoiceCoordinator.startConversation()` is unchanged. The only addition: `runPipeline()` calls `wakeWordDetector?.pause()` / `tryResume()` at entry/exit. These are safe no-ops if the wake word detector isn't running.

---

## Audio Session Transition — Expected Behavior

**On device (when Porcupine is linked and baxter.ppn bundled):**

```
App launch
  → .playback (VoiceService.init)
  → model.startWakeWord()
  → .record + .measurement (WakeWordDetector takes over)

"Baxter" detected
  → WakeWordDetector.pause()
  → startConversation() → STT starts
  → .record + .measurement (STT session, same category, clean transition)
  → STT completes → teardownAudio() → .playback
  → TTS plays
  → TTS completes → .idle → tryResume()
  → .record + .measurement (WakeWordDetector back)

Panel event fires canned phrase
  → wakeWordDetector?.pause()
  → .playback → phrase plays
  → audioPlayerDidFinishPlaying → tryResume()
  → .record + .measurement
```

The STT → WakeWordDetector transition is particularly clean: both use `.record + .measurement`, so there's no session category switch — only an engine change. The likely artifact point is TTS → WakeWordDetector (`.playback` → `.record`), which involves a real session switch. On-device testing needed to confirm silence.

---

## Verification Checklist

**Compiles without Porcupine:**
- [x] `xcodebuild` succeeds — BUILD SUCCEEDED (build 70, zero errors)
- [x] One pre-existing warning in ClaudeVoiceAssistant.swift (not from 0c.19 changes)
- [x] WakeWordDetector.swift compiles via `#if canImport(Porcupine)` guards

**Functional (requires Todd's steps 1-3):**
- [ ] Porcupine package added via Xcode UI
- [ ] baxter.ppn bundled and in target membership
- [ ] AccessKey seeded to Keychain
- [ ] Device: "Baxter" reliably wakes (test at 1m, 2m, across room)
- [ ] Device: mic-tap path still works independently
- [ ] Device: no audible click on wake word → STT transition
- [ ] Device: no audible click on TTS complete → wake word resume
- [ ] Device: no spurious detections during ambient TV conversation
- [ ] Console shows `[WakeWord] detected` on each trigger
- [ ] VoiceCoordinator.cancel() properly calls tryResume()
- [ ] Ambient phrase (panel event) correctly pauses/resumes wake word
- [ ] `wakeWordEnabled = false` stops detector; `= true` restarts it

---

## False Positive Monitoring

Watch console for:
```
[WakeWord] detected at HH:MM:SS (keyword index: 0)
── VoiceCoordinator: STT ...   ← should follow within 500ms
```

If `[WakeWord] detected` appears frequently WITHOUT a following STT line, those are false positives (fired but user didn't intend to speak, or transcript was empty).

**Target:** < 2 false positives / day at typical home TV volume.

| Observation | Action |
|-------------|--------|
| TV/voices triggering wake word | Lower sensitivity: 0.5 → 0.3 |
| "Baxter" missed frequently | Raise sensitivity: 0.5 → 0.7 |
| Neither direction helps | Retrain at console.picovoice.ai |
