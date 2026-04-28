# Home Control Wall Panel — Project Audit
**Date:** 2026-04-28 | **Build on `main`:** 71 (compiled) | **Build on iPad:** 70

---

## What's Been Built

### The Physical Thing
An always-on iPad Air (M2, 13") wall-mounted home control panel. Landscape, full-screen, no status bar. Shows live Home Assistant data, ambient photos from iCloud, and has a voice AI (Baxter) you talk to. Private Swift/SwiftUI app, local repo only.

---

### Layer 1 — HA Data + UI Shell *(Phases 0a–0c.15)*

- **Live polling** from Home Assistant via REST API every 10 seconds
- **StatusBar** (top strip): time, WeatherKit conditions, home mode (Home/Away/Sleep), door/garage/lock indicators
- **Climate display**: current/target/humidity trinity, HVAC mode arrow, +/− controls wired to UI
- **Lighting panel**: room grid, per-room drill-in with individual bulbs — entity state from HA
- **Ambient photo display**: cycles through iCloud album when idle
- **Settings drill-in**: HA URL, connection state, preferences
- All controls show a **write-disabled toast** when tapped — no actual HA writes happen yet

---

### Layer 2 — Voice Pipeline *(Phases 0c.16–0c.20)*

End-to-end streaming voice conversation. Every stage streams:

```
Mic tap (or wake word)
  → Apple STT (SFSpeechRecognizer) — on-device, no cloud
  → Claude API (streaming SSE)     — token-by-token
  → Cartesia WebSocket TTS          — audio chunks back while LLM still generating
  → AVAudioEngine streaming         — plays audio as chunks arrive, no wait for full response
```

**Working:**
- Mic tap → full conversation round-trip → streaming audio response
- STT with VAD auto-finalize (silence detection ends the utterance)
- LLM via Anthropic streaming API, 6-turn conversation history in memory
- TTS via Cartesia WebSocket, Benedict voice, 0.8× speed
- **Thinking tick** audio plays during LLM latency gap (30 phrases, 200–229) ← *wired correctly in build 71*
- **Ambient panel phrases** play from the wall panel for atmosphere (170 phrases across pools)
- TranscriptBand shows user utterance + AI response as text overlaid on panel
- Voice modal overlay during active conversation
- **System prompt** (Baxter v2) — full character brief now in bundle for the first time in build 71 (all prior builds used a minimal fallback)

**Baxter's character (v2):**
- British, measured, dry wit — addresses Todd as "sir", Mei-Ling as "ma'am", Cora as "Miss Cora"
- Backronym: *"Baxter, sir. Baritone Aide of Xenial Temperament, Ever Ready."*
- Wit restraint: plain by default, wit is rare and therefore lands
- Length: 1–3 sentences for commands; full depth honored when you ask for a story or explanation

---

### Layer 3 — Wake Word *(Phase 0c.19 — infrastructure complete, activation pending)*

`WakeWordDetector.swift` fully implemented:
- Picovoice Porcupine integration behind `#if canImport(Porcupine)` — compiles and runs without the package linked
- Full audio session lifecycle: pauses during STT/TTS, resumes after
- Sensitivity constant at `WakeWordDetector.sensitivity = 0.5` (file: `Services/Voice/WakeWordDetector.swift`, line ~54)
- On "Baxter" detected → identical pipeline to mic tap

**Status: silent no-op.** Three manual steps required before it activates (see Outstanding §4).

---

### Infra

- **Keychain** storage for all API keys (Cartesia, Anthropic, Picovoice)
- **`[Baxter]` telemetry** — every pipeline event logs with this one searchable prefix (see Telemetry Guide below)
- **Handoff repo** (`github.com/tlavigne974os/home-control-handoff`) — public, phase completion reports + character files
- **Character versioning** — `tools/sync-character.sh` mirrors system prompt versions to this repo
- **WeatherKit** — live conditions and hourly forecast

---

## Current Build State

| Item | State |
|---|---|
| Build on `main` | **71** — compiled, NOT on iPad |
| Build on iPad | **70** — missing: thinking ticks, `[Baxter]` telemetry, system prompt v2 |
| App display name | "Home Control" |
| Voice assistant name | Baxter |

**To push build 71 to iPad** (plug in USB first):
```bash
cp -R ~/Library/Developer/Xcode/DerivedData/Build/Products/Debug-iphoneos/WallPanel.app /tmp/WallPanel.app
xcrun devicectl device install app --device 25423385-294F-50EA-8498-5690FECE21BF /tmp/WallPanel.app
```

---

## Outstanding / Known Issues

### 🔴 Must-fix before next feature

**1. HA writes still disabled — the whole control layer doesn't work**
Every button/slider/toggle shows a write-disabled toast. `callService()` exists in `PollingHAClient` but is never called. This is the biggest gap between what the panel *looks* like it does and what it actually does. Before voice can control lights or climate, the write layer needs to be enabled.

**2. System prompt identity paragraph still says "Woadhouse"**
`baxter-system-prompt-v2.md` opens with "You are Woadhouse." The name handling section correctly says Baxter, but the identity paragraph and examples still reference Woadhouse. Should be cleaned up as a v3 pass.

**3. Build 71 not on device**
Build 70 is running. Build 71 has: thinking ticks properly bundled, `[Baxter]` telemetry, system prompt v2. Needs a USB install.

---

### 🟡 Pending — Todd manual steps

**4. Wake word activation (3 steps)**
- Add Porcupine SPM via Xcode UI: **File → Add Package Dependencies** → `https://github.com/Picovoice/porcupine` — one-time ~3GB download, shows a progress bar
- Download `baxter.ppn` from [console.picovoice.ai](https://console.picovoice.ai/) → copy to `WallPanel/WallPanel/Audio/baxter.ppn` → add to Xcode target membership
- Seed AccessKey to Keychain: add the following to `WallPanelApp.init()`, build once, then remove and build again:
  ```swift
  KeychainHelper.save(key: "VoiceAPI.picovoiceKey", value: "YOUR-KEY")
  ```

Until done, wake word is a silent no-op. Mic tap works fine.

**5. Thinking tick audio curation**
Phrase 200–229 are in the bundle as of build 71. Worth auditioning all 30 — Cartesia rendering can be inconsistent on short phrases. Re-render duds via the render script in `tools/`.

---

### 🟡 Product gaps — future phases

**6. Voice can't actually control the home**
Baxter deflects all control requests ("my hands are largely metaphorical"). The path: HA writes enabled → Claude tool-use layer → `callService()`. Not started.

**7. Baxter doesn't know current home state**
Claude gets no HA context. "Is the kitchen light on?" gets a deflection. Need to inject current home state into the system prompt or via tool-use before voice control is useful.

**8. Conversation history resets on every app launch**
6-turn history is in-memory only. No cross-session persistence. Acceptable for a wall panel; Baxter just forgets everything on reboot.

**9. No Settings UI for wake word toggle**
`wakeWordEnabled: Bool` exists in AppModel (with a `didSet` that starts/stops the detector) but there's no Settings toggle wired to it. Runtime-only for now.

**10. Porcupine sensitivity not tunable from UI**
`static let sensitivity: Float32 = 0.5` in `WakeWordDetector.swift` line ~54. Code edit required. Worth a Settings entry once wake word is live and you're calibrating false positives.

**11. False positive baseline unknown**
Wake word has never run in lived use. Spec target: <2 false positives/day at typical home TV volume. Won't know if that's achievable until Porcupine is linked.

---

### 🟢 Polish

**12. "Woadhouse" in iOS permission strings**
`Info.plist` `NSMicrophoneUsageDescription` and `NSSpeechRecognitionUsageDescription` still say "Woadhouse uses the microphone / speech recognition…" — shown in iOS permission dialogs. Low stakes (shown once at first launch) but should say "Baxter" or "Home Control."

**13. Token count missing from streaming telemetry**
`VoiceCoordinator` has `// TODO: capture from streaming usage event` for `inputTokens`. Usage stats from streaming API aren't captured. Fine now, useful later for cost tracking.

**14. Mac Catalyst deployment not set up**
iPad apps run natively on M1/M2 Macs via Mac Catalyst. Would make development and screenshots much faster. Not attempted yet.

**15. Ambient phrases not categorized in phrases.json**
All 199 entries have `category: "?"`. Pool assignment is hardcoded by number range in `VoiceService.poolNumbers`. Not broken, just messy.

---

## Architecture Snapshot

```
AppModel (Observable)
  ├── HAController            — HA polling, home state, write toast
  ├── WeatherService          — WeatherKit, forecast
  ├── VoiceService            — ambient phrases, audio session owner
  ├── VoiceCoordinator        — pipeline orchestrator (STT→LLM→TTS)
  │     ├── AppleSpeechCoordinator  — Apple STT
  │     ├── ClaudeVoiceAssistant    — LLM, streaming
  │     │     └── AnthropicAPIClient
  │     └── AudioStreamPlayer       — streaming playback
  ├── WakeWordDetector        — Porcupine (inactive until linked)
  └── CartesiaWSClient        — WebSocket TTS (active)

Keys in Keychain:
  VoiceAPI.anthropicKey     ✅ seeded
  VoiceAPI.cartesiaKey      ✅ seeded
  VoiceAPI.picovoiceKey     ❌ pending (Todd action)

Audio in bundle (build 71):
  phrase_001–170.m4a   ✅  ambient pool (170 phrases)
  phrase_200–229.m4a   ✅  thinking ticks (30 phrases)
  baxter.ppn           ❌  pending (download from Picovoice Console)
```

---

## Telemetry Guide (build 71+)

**Console.app on Mac → filter process: `WallPanel` → search: `[Baxter]`**

```
[Baxter] system prompt loaded (2847 chars)    ← full prompt loaded ✅  (not fallback)
[Baxter] pool thinkingTick: 30/30 files found ← ticks in bundle ✅
[Baxter] pool thinkingTick: 0/30 files found  ← build 70 or earlier — ticks missing
[Baxter] system prompt file not found — using fallback  ← any build before 71

[Baxter] listening started                    ← STT engaged
[Baxter] stop → transcript = "..."           ← what STT heard
[Baxter] thinking tick playing (duration: Xs) ← tick fires, covers LLM latency
[Baxter] LLM first token received            ← Claude API latency ends here
[Baxter] first Cartesia audio chunk received  ← TTS latency ends here

[Baxter] wake word detected at HH:MM:SS      ← Porcupine fired (once linked)
```

**Diagnosing slow voice:** timestamp delta between `listening started` → `LLM first token` = STT + API latency. Delta between `LLM first token` → `first Cartesia audio chunk` = TTS latency. If `awaiting tick completion...` appears, the thinking tick is the bottleneck (it's longer than the total LLM+TTS round trip).

---

## Completion Reports

All phase reports are in this repo under `completions/`:

| Phase | Report | Build |
|---|---|---|
| 0c.18 — Streaming pipeline | `2026-04-28-phase-0c-18-streaming.md` | 69 |
| 0c.19 — Wake word | `2026-04-28-phase-0c-19-wakeword.md` | 70 |
| 0c.20 — System prompt v2 | `2026-04-28-phase-0c-20-system-prompt-v2.md` | 71 |
| Session handoff (DC) | `2026-04-28-session-handoff-dc.md` | 71 |
| **This document** | `2026-04-28-project-audit.md` | 71 |

Character files and versioning history: [`baxter-character/`](../baxter-character/)
