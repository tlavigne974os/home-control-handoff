# Phase 0c-23 Completion — Mac Catalyst Voice Debugging

**Build:** 75 (0.5.4)  
**Date:** 2026-04-29  
**Installed:** iPad Pro (8B58895C) ✓, iPad Air (25423385) ✓  

---

## Changes Made

### 1. SUPPORTS_MACCATALYST = YES (WallPanel.xcodeproj)
Added to both Debug and Release build configurations:
```
SUPPORTS_MACCATALYST = YES;
TARGETED_DEVICE_FAMILY = "1,2";
```

### 2. Porcupine xcframework excluded from Mac Catalyst link phase
`PvPorcupine.xcframework` has no Mac slice. Added `platformFilter = ios` to the build file:
```
5FBB5BBF2FA1472E00229437 /* Porcupine in Frameworks */ = {
    isa = PBXBuildFile;
    platformFilter = ios;   ← added
    productRef = 5FBB5BBE2FA1472E00229437 /* Porcupine */;
};
```
WakeWordDetector.swift already uses `#if canImport(Porcupine)` throughout — no source changes needed.

### 3. UIApplication.isIdleTimerDisabled guarded for Mac Catalyst
`WallPanelApp.swift`:
```swift
.onAppear {
    #if !targetEnvironment(macCatalyst)
    UIApplication.shared.isIdleTimerDisabled = true
    #endif
}
```

### 4. No changes needed to PhotoLibrary.swift
`Photos` and `UIKit` are available on Mac Catalyst. Degraded ambient view (no iCloud album) is acceptable per DC.

---

## Mac Catalyst Build Result

```
** BUILD SUCCEEDED **
```
Only pre-existing Swift 6 concurrency warning (ClaudeVoiceAssistant, not a new issue).

---

## Mac Catalyst Runtime — Console Output Captured

App launched from DerivedData with stdout capture (`stdbuf -oL`). Full output:

```
[Baxter] pool canonMain: 109/109 files found
[Baxter] pool transitionToQuick: 10/10 files found
[Baxter] pool casualMic: 30/30 files found
[Baxter] pool transitionToFull: 10/10 files found
[Baxter] pool transitionToAmbient: 10/10 files found
[Baxter] pool thinkingTick: 30/30 files found
[Baxter] loaded 199 phrase texts for transcript display
[Baxter] system prompt v3 loaded (5663 chars)
[Baxter] wake word disabled — skipping start
[Baxter] 🔄 presentationState: idle → listening
[Baxter] startListening() — sessionId=43702F3E
```
*(app terminated here — SIGABRT)*

**Notes on the output:**
- Audio phrase pools loaded correctly ✓
- System prompt loaded ✓
- Wake word disabled on Mac ✓ (Porcupine is iOS-only, guarded)
- Mic button found via AX API at screen pos (1180, 743), successfully pressed ✓
- `presentationState: idle → listening` ✓ (button registered, state machine works)
- `startListening() — sessionId=...` ✓ (new session started, session ID minted)

---

## Mic-Tap Crash — Root Cause

App crashes inside `startListening()` at `try engine.start()` (AppleSpeechCoordinator.swift line 127).

**Crash stack (from iOS Simulator crash report — same abort pattern confirmed on Mac Catalyst):**
```
AppleSpeechCoordinator.startListening()  [line 127]
  → -[AVAudioEngine startAndReturnError:]
  → AVAudioEngineGraph::Start()
  → AURemoteIO::Start()
  → AURemoteIO::fetchWorkgroup()
  → _ReportRPCTimeout()
  → abort() → SIGABRT
```

**This is the same `AURemoteIO::fetchWorkgroup()` timeout as the iOS Simulator.**

On Mac Catalyst, the cause is: app launched from DerivedData without TCC microphone permission. The `com.tramel.homecontrol.wallpanel` bundle is not in System Settings > Privacy > Microphone (only appears for apps installed in /Applications). When `engine.start()` tries to access the audio hardware workgroup, the audio server refuses (no mic permission) → RPC timeout → abort.

**System Settings > Privacy > Microphone shows:** ChatGPT.app, claude.app, Microsoft Teams.app, zoom.us.app — but NOT `Home Control`. The app has never been granted microphone access on this Mac.

---

## Important Distinction: Mac vs. iPad Behavior

The Mac Catalyst SIGABRT is a **different failure mode** from the iPad "instant-fail":

| | Mac Catalyst | iPad "instant-fail" |
|---|---|---|
| Result | SIGABRT crash | Returns to idle with empty transcript |
| Crash | `AURemoteIO::fetchWorkgroup` abort | No crash |
| Cause | No TCC mic permission | Unknown — needs real-device Console log |
| Code path | Crashes at `engine.start()` | Presumably completes `startListening()` |

The Mac cannot tell us why the iPad instant-fails — they're different failure modes.

---

## To Fix Mac Catalyst Mic Access

Option A (recommended for next session): Run from Xcode debugger with the Mac Catalyst destination. Xcode registers the app with TCC properly.

```bash
xcodebuild -project WallPanel/WallPanel.xcodeproj \
  -scheme WallPanel \
  -destination 'platform=macOS,variant=Mac Catalyst' \
  -allowProvisioningUpdates \
  run
```

Option B: Grant mic permission manually. The TCC dialog should appear when `engine.start()` is called if the app is launched with proper macOS identity (e.g., via `open` from Finder). The dialog wasn't attributed to `Home Control` when launched from Terminal.

---

## Git Commit

`eee268b` — "Phase 0c-23: enable Mac Catalyst for voice debugging (build 75)"

## Build 75 Deployed

- iPad Pro (8B58895C): ✓ installed at 10:44:34
- iPad Air (25423385): ✓ installed at 10:44:51
