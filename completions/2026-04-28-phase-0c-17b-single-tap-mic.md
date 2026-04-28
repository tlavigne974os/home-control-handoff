# Phase 0c.17b — Single-tap mic + tap-steal fix
**Build 68 · 2026-04-28**

---

## What was broken (build 67)

Three user-reported issues discovered during first live use of the Phase 0c.17 conversational pipeline:

1. **Hold-to-listen not working** — `onLongPressGesture` on the mic icon was silently swallowed. The parent `StatusBar` view applied `.contentShape(Rectangle()).onTapGesture` at the same level, causing the parent to win simultaneous gesture recognition. The long-press threshold never fired.

2. **Tapping mic opens quick control** — Same root cause. Every tap on any of the three right-hand icons (front door, garage, mic) fired the parent `.onTapGesture { openQuickControl() }` simultaneously with the icon's own gesture.

3. **Hold model removed by user request** — User asked to drop the 400ms hold threshold entirely. "Just tap the mic, that's what the picture is." Single tap should start listening immediately.

---

## Changes

### `VoiceService.swift`
- Added `case listening` to `VoicePresentationState` enum
- Added `case (.listening, .listening): return true` to `Equatable` conformance

### `AppleSpeechCoordinator.swift`
- Added `waitForAutoFinalize() async -> String` — VAD path that suspends on a `CheckedContinuation` without stopping the audio engine. Resumes when `isFinal` fires (or 10s max-duration guard). The prior `stopListening()` was a hybrid that tore down audio immediately then waited; this new method lets recognition run until the engine itself fires end-of-speech.

### `VoiceCoordinator.swift`
- Replaced `startListening()` + `stopAndSend()` with a single `startConversation()` entry point
- `startConversation()` sets `.listening` state, fires light haptic, cancels any in-flight task, starts a new `runPipeline()` task
- `runPipeline()` now calls `speechCoordinator.startListening()` internally, then awaits `waitForAutoFinalize()` (VAD)
- Removed all hold-threshold logic

### `AppModel.swift`
- Replaced `startVoiceListening()` + `stopAndSendVoice()` with `startConversation()`

### `StatusBarView.swift`
- **Tap-steal fix**: Moved `openQuickControl` tap gesture from `.contentShape(Rectangle()).onTapGesture` (foreground, fires simultaneously with child buttons) to `.background(Color.clear.contentShape(Rectangle()).onTapGesture { … })` (background layer, only fires when no foreground button intercepts)
- **MicButton rewrite**: Removed `onLongPressGesture` entirely. Replaced with a plain `Button { model.startConversation() }`. Color is `hearthAmber` when `presentationState == .listening`, `hearthAmberDim` otherwise.

### `VoiceModalOverlay.swift`
- Added `presentationState: VoicePresentationState` parameter
- `formState` is now driven by `presentationState` transitions via `.onChange`:
  - `.listening` → `VoiceFormState.listening` (outward rings — user is speaking)
  - `.processingUserUtterance` → `.thinking` (breathing pulse — LLM in flight)
  - `.modalSpeaking` → `.speaking` (inward rings — audio playing)
- `onAppear` seeds the initial `formState` from the current `presentationState`:
  - Pre-rendered phrase path (`.modalSpeaking` on appear): starts in `.thinking`, transitions to `.speaking` after 300ms — preserving the existing Woadhouse "thinking pause" behavior
  - Conversational path (`.listening` on appear): starts in `.listening` immediately

### `RootView.swift`
- Added `showVoiceModal: Bool` computed property — true for `.listening`, `.processingUserUtterance`, `.modalSpeaking`
- `VoiceModalOverlay` now shown for all three states (not just `.modalSpeaking`)
- Overlay stays in the view hierarchy across the three-state pipeline, so `VoiceForm` ring animations are continuous (no remove/re-insert between phases)
- Passes `presentationState` to `VoiceModalOverlay`

### `TranscriptBand.swift`
- Added `case .listening: break` to the exhaustive switch — band stays hidden during listening (the VoiceModalOverlay handles that visual)

---

## Architecture note — VAD vs push-to-talk

The old press-and-hold model was push-to-talk (PTT): user holds → STT active → user releases → pipeline fires. The new model is **voice activity detection (VAD)**: user taps → STT active → recognizer fires `isFinal` automatically when silence is detected → pipeline fires with no second user action.

`SFSpeechRecognizer` with `requiresOnDeviceRecognition = true` is reliable for VAD at conversational lengths. The 10-second `maxDurationTask` in `AppleSpeechCoordinator` acts as a safety net if the recognizer stalls.

---

## Verification checklist

| # | Scenario | Expected |
|---|----------|----------|
| 1 | Tap mic in ambient mode | Quick control does NOT open; `.listening` state → outward rings |
| 2 | Tap front door icon | Front door action fires; quick control does NOT open |
| 3 | Tap garage icon | Garage action fires; quick control does NOT open |
| 4 | Tap empty status bar area | Quick control opens normally |
| 5 | Tap mic, speak naturally | Ring animation → transcript band shows user's words → thinking pulse → Woadhouse responds |
| 6 | Tap mic, say nothing | 10s timeout → silence → idle (no crash, no band) |
| 7 | Tap mic while Woadhouse is speaking | Old pipeline cancelled, new `.listening` starts |
| 8 | Tap dim overlay during response | `onCancel()` fires → idle, overlay dismisses |
| 9 | Pre-rendered phrase (panel event) | `.modalSpeaking` → overlay with thinking→speaking transition (unchanged) |
| 10 | Conversational full round-trip | listening → processing (user text) → speaking (Woadhouse) → idle |

---

## Files changed
```
WallPanel/WallPanel/AppModel.swift
WallPanel/WallPanel/BuildInfo.swift
WallPanel/WallPanel/Info.plist
WallPanel/WallPanel/Services/Voice/AppleSpeechCoordinator.swift
WallPanel/WallPanel/Services/Voice/VoiceCoordinator.swift
WallPanel/WallPanel/Services/VoiceService.swift
WallPanel/WallPanel/Views/RootView.swift
WallPanel/WallPanel/Views/StatusBarView.swift
WallPanel/WallPanel/Views/Voice/TranscriptBand.swift
WallPanel/WallPanel/Views/Voice/VoiceModalOverlay.swift
```
