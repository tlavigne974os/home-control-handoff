# Phase 0c-24 Complete — Mac Catalyst Voice Pipeline Verified

**Date:** 2026-04-29  
**Phase:** 0c-24 — Mac Catalyst mic TCC + full pipeline test

---

## Result: PASS

Full STT → Claude → Cartesia → audio pipeline working on Mac Catalyst (macOS 26.4.1 beta).

### Verified interaction

**User said:** "Hey what time is it Baxter what time is it Baxter"

**STT:** first partial at 2.43s ("Hey")  
**LLM TTFT:** 522ms  
**Baxter replied:** "Right then. I haven't been given access to the current time, sir. You'll need to check your phone or a..."  
**Audio:** played and stopped ✅

---

## Fixes applied in this phase

### 1. `requestPermissions()` wired into pipeline
Was dead code — existed in `AppleSpeechCoordinator` but never called.  
Fixed: `await speechCoordinator.requestPermissions()` added to `VoiceCoordinator.runPipeline()`.

### 2. Mac mic TCC — `AVCaptureDevice` vs `AVAudioSession`
`AVAudioSession.requestRecordPermission` returns `false` silently on Mac Catalyst without dialog.  
Fixed: `#if targetEnvironment(macCatalyst)` branch uses `AVCaptureDevice.requestAccess(for: .audio)`.

### 3. Hardened Runtime entitlement
Without `com.apple.security.device.audio-input`, `AVCaptureDevice.requestAccess` returns `false` on macOS 26 without showing any TCC dialog.  
Fixed: added `<key>com.apple.security.device.audio-input</key><true/>` to `WallPanel.entitlements`.

### 4. Log capture via `freopen()`
`open`-launched apps don't forward stdout to terminal. `stdbuf` breaks TCC bundle identity (crash).  
Fixed: `freopen("/tmp/baxter-mac.log", "w", stdout)` in `WallPanelApp.init()` under macCatalyst.

### 5. Network STT on Mac Catalyst
On-device STT model returns empty transcript silently on macOS 26 beta even with audio flowing.  
Fixed: `requiresOnDeviceRecognition = false` for `targetEnvironment(macCatalyst)`.

### 6. Mac Catalyst keychain bootstrap
Keys seeded on iOS device are in iOS keychain — not accessible to Mac Catalyst app.  
`security add-generic-password` adds to macOS keychain but requires per-item access dialog per app.  
Fixed: `WallPanelApp.init()` reads `~/.baxter-mac-keys` on Mac Catalyst startup, seeds keychain using app's own `KeychainHelper.save()` (app-owned items: no access dialog). One-time setup.

---

## Mac dev launch procedure (documented)

```bash
# Build
cd ~/Projects/home-control/WallPanel
xcodebuild -project WallPanel.xcodeproj -scheme WallPanel \
  -destination 'platform=macOS,variant=Mac Catalyst' \
  -configuration Debug build

# Launch (always via open — stdbuf breaks TCC)
open ~/Library/Developer/Xcode/DerivedData/Build/Products/Debug-maccatalyst/pr.app

# Watch logs
tail -f /tmp/baxter-mac.log

# Trigger mic programmatically
/tmp/press_mic
```

### First-time key setup (once per Mac)
```
# Create ~/.baxter-mac-keys:
ANTHROPIC_API_KEY=sk-ant-...
CARTESIA_API_KEY=<uuid>
```
Launch the app once — it seeds the keys into its own keychain and they persist forever.

---

## Known remaining issue

Cartesia TTS streaming gets a "bad response from the server" after partial audio delivery — response is truncated mid-sentence. Audio does play, pipeline does run. This is a Cartesia WebSocket streaming issue to investigate in a follow-on phase.

---

## macOS 26 beta quirks discovered

- `AVAudioSession.requestRecordPermission` → silent `false` (no dialog, no error)
- `AVCaptureDevice.requestAccess(for: .audio)` → correct mic TCC dialog (requires `com.apple.security.device.audio-input` entitlement)
- On-device SFSpeechRecognizer → empty transcript (network recognition works)  
- `launchctl setenv` → does NOT propagate to GUI apps launched via `open`
- `security add-generic-password` → per-app access dialog required; app-owned keychain items do not need dialogs
- `stdbuf` launch → `__TCC_CRASHING_DUE_TO_PRIVACY_VIOLATION__` (breaks bundle identity for TCC)
