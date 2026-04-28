# Session Handoff — 2026-04-28 (CC → DC)

**Session type:** Full implementation session (phases 0c.19 + 0c.20)  
**App repo:** `~/Projects/home-control` (private, local only — no remote)  
**Handoff repo:** `https://github.com/tlavigne974os/home-control-handoff`  
**Status:** Pausing for audit. Both phases committed to main. Device needs build 71 installed.

---

## What Shipped This Session

### Phase 0c.19 — Wake Word "Baxter" via Picovoice Porcupine

**Branch:** `phase-0c-19-wakeword` — **merged to main** (build 70)  
**Tag:** `v0.8.0-wakeword`  
**Completion report:** `completions/2026-04-28-phase-0c-19-wakeword.md`

New class `WakeWordDetector` — full audio session lifecycle management, audio conversion (44100Hz Float32 → 16kHz Int16 via AVAudioConverter), Porcupine integration behind `#if canImport(Porcupine)` guards. App builds and runs normally without Porcupine linked — wake word just silently disabled.

Audio session coordination:
- Wake word pauses at every pipeline entry point (VoiceCoordinator.runPipeline, VoiceService panel events)
- Resumes via `tryResume()` at every pipeline exit (idle return, TTS complete, cancel)
- Two AVAudioEngine instances cannot co-exist — pause() fully stops the engine (not just halts tap)

**Todd still needs to do before wake word activates:**
1. Add Porcupine via Xcode UI: File → Add Package Dependencies → `https://github.com/Picovoice/porcupine`
2. Download `baxter.ppn` from Picovoice Console → copy to `WallPanel/WallPanel/Audio/baxter.ppn` → add to Xcode target
3. Seed AccessKey to Keychain: add `KeychainHelper.save(key: "VoiceAPI.picovoiceKey", value: "YOUR-KEY")` to `WallPanelApp.init()`, build once, remove the seed code, build again

---

### Phase 0c.20 — Baxter System Prompt v2 + Character Versioning

**Branch:** main directly (committed as configuration, not a feature branch)  
**Build:** 71  
**Completion report:** `completions/2026-04-28-phase-0c-20-system-prompt-v2.md`

Three character calibrations applied:

| Section | Change |
|---------|--------|
| `NAME HANDLING` | Baxter backronym: "Baritone Aide of Xenial Temperament, Ever Ready" + self-aware tag variants |
| `LENGTH` | Expanded: honors stories/explanations when invited; brevity rule scoped to command-and-response |
| `CHARACTER CALIBRATION — RESTRAINT` | New section: wit restraint discipline, vocabulary limits (any one "purview/indeed/rather" at most once per conversation), if-witty-last-response-be-plain-next rule |

**Critical discovery:** `woadhouse-system-prompt-v1.md` was never in the Resources build phase — all prior builds used the minimal fallback prompt. Build 71 is the FIRST build where the full system prompt lands in the bundle and Baxter actually has his full character.

Versioning infrastructure:
- `WallPanel/WallPanel/Resources/baxter-system-prompt-v1.md` — canonical v1 copy (sha256 verified identical to original)
- `WallPanel/WallPanel/Resources/baxter-system-prompt-v2.md` — three edits applied
- `tools/sync-character.sh` — mirrors `Resources/baxter-system-prompt-v*.md` to public handoff repo's `baxter-character/` directory
- `home-control-handoff/baxter-character/` — README + both versions now in public repo

Future character iteration loop:
1. Edit/create `Resources/baxter-system-prompt-vN.md`
2. Update `ClaudeVoiceAssistant.loadSystemPrompt()` resource name
3. Test in lived use
4. `cd ~/Projects/home-control && bash tools/sync-character.sh`

---

### Telemetry Unification (bonus, same session)

All `print()` statements across the voice pipeline now use `[Baxter]` as the single searchable prefix.

**How to see telemetry:** Console.app (Mac) → filter process `WallPanel` → search `[Baxter]`

**Sample timeline:**
```
[Baxter] system prompt loaded (2847 chars)
[Baxter] pool thinkingTick: 0/30 files found   ← expected; phrase_200-229.m4a don't exist
[Baxter] listening started
[Baxter] stop → transcript = "..."
[Baxter] thinking tick playing (duration: X.XXs)
[Baxter] LLM first token received
[Baxter] first Cartesia audio chunk received
```

---

## Known Issues / Gaps for Next DC Session

### 1. Build 71 not on device (HIGH)
Build 70 is running on the iPad. Build 71 is compiled and in DerivedData but not installed — iPad was unavailable (off USB) at session end.

**To install:**
```bash
cp -R ~/Library/Developer/Xcode/DerivedData/Build/Products/Debug-iphoneos/WallPanel.app /tmp/WallPanel.app
xcrun devicectl device install app --device 25423385-294F-50EA-8498-5690FECE21BF /tmp/WallPanel.app
```

### 2. Thinking ticks missing (MEDIUM)
`phrase_200-229.m4a` files do not exist in the bundle. The thinking tick pool (`PhrasePool.thinkingTick: Array(200...229)`) is always empty at runtime. No audio plays during the LLM latency gap. Console shows `[Baxter] pool thinkingTick: 0/30 files found` at launch.

**Fix:** Record or synthesize 30 short "thinking" audio clips (750ms–2s), name them `phrase_200.m4a` through `phrase_229.m4a`, add to `Audio/` directory in Xcode target.

**Alternatively:** Remap the thinkingTick pool to existing files (e.g., very short ambient phrases) — all code is already correct, just need audio files in range 200-229.

### 3. Porcupine wake word (MEDIUM — Todd action needed)
Wake word is fully implemented but inactive. Requires Todd to:
- Add Porcupine SPM package via Xcode UI (3GB download, one-time)
- Download `baxter.ppn` from Picovoice Console
- Seed AccessKey to Keychain

### 4. Voice latency investigation (LOW)
User reported voice responses feel slower in build 70 vs earlier. Most likely culprit: Stage 4 (`await tick completion`) blocks audio playback if the thinking tick is longer than LLM+TTS latency. Second possible cause: system prompt is now actually loading (v2 at ~2800 chars vs fallback at ~200 chars) which slightly increases token count per call.

Telemetry to collect: time from `[Baxter] listening started` → `[Baxter] LLM first token received` → `[Baxter] first Cartesia audio chunk received` → audio plays.

---

## App State at Handoff

| Item | State |
|------|-------|
| App repo branch | `main` |
| Build on main | 71 (compiled, NOT installed) |
| Build on iPad | 70 |
| Porcupine | Not linked (Todd adds via Xcode UI) |
| Wake word active | No (Porcupine not linked) |
| System prompt | v2 (first build with full prompt in bundle) |
| Telemetry prefix | `[Baxter]` everywhere |
| Phase 0c.18 | ✅ merged to main, completion report pushed |
| Phase 0c.19 | ✅ merged to main, completion report pushed |
| Phase 0c.20 | ✅ committed to main, completion report pushed |

---

## Files Changed This Session

### App repo (`~/Projects/home-control/`)

| File | Change |
|------|--------|
| `WallPanel/WallPanel/Services/Voice/WakeWordDetector.swift` | **NEW** — 0c.19 |
| `WallPanel/WallPanel/Services/Voice/VoiceAPIConfig.swift` | picovoiceKeyName + picovoiceAccessKey — 0c.19 |
| `WallPanel/WallPanel/Services/Voice/VoiceCoordinator.swift` | wakeWordDetector ref + pause/resume — 0c.19; `[Baxter]` telemetry |
| `WallPanel/WallPanel/Services/VoiceService.swift` | wakeWordDetector ref + pause/resume — 0c.19; `[Baxter]` telemetry |
| `WallPanel/WallPanel/Services/Voice/AppleSpeechCoordinator.swift` | `[Baxter]` telemetry |
| `WallPanel/WallPanel/Services/Voice/ClaudeVoiceAssistant.swift` | loads baxter-system-prompt-v2 — 0c.20; `[Baxter]` telemetry |
| `WallPanel/WallPanel/AppModel.swift` | WakeWordDetector ownership + startWakeWord() — 0c.19; `[Baxter]` telemetry |
| `WallPanel/WallPanel/Views/RootView.swift` | startWakeWord() in onAppear — 0c.19 |
| `WallPanel/WallPanel/Resources/baxter-system-prompt-v1.md` | **NEW** — 0c.20 |
| `WallPanel/WallPanel/Resources/baxter-system-prompt-v2.md` | **NEW** — 0c.20 |
| `WallPanel/WallPanel/Info.plist` | build 69 → 70 → 71 |
| `WallPanel/WallPanel.xcodeproj/project.pbxproj` | WakeWordDetector.swift + baxter-system-prompt v1/v2 |
| `tools/sync-character.sh` | **NEW** — 0c.20 |

### Handoff repo (`~/Projects/home-control-handoff/`)

| File | Change |
|------|--------|
| `completions/2026-04-28-phase-0c-19-wakeword.md` | **NEW** — pushed |
| `completions/2026-04-28-phase-0c-20-system-prompt-v2.md` | **NEW** — pushed |
| `completions/2026-04-28-session-handoff-dc.md` | **NEW** — this file |
| `baxter-character/README.md` | **NEW** — pushed |
| `baxter-character/baxter-system-prompt-v1.md` | **NEW** — pushed |
| `baxter-character/baxter-system-prompt-v2.md` | **NEW** — pushed |

---

## Recommended Audit Focus

1. **Voice latency** — use `[Baxter]` telemetry in Console to measure each pipeline stage. Compare STT latency, LLM TTFT, Cartesia first-chunk, audio start against your expectations.

2. **System prompt v2 in lived use** — now that the full prompt actually loads, evaluate: Does Baxter feel more calibrated? Is the wit restraint working? Does "tell me a story" get honored at full length?

3. **Thinking tick gap** — conversations now have silence during LLM latency (since ticks don't have audio files). Worth deciding: synthesize tick audio, or pull silence as acceptable?

4. **Porcupine** — when ready to activate wake word, follow Step 1 instructions in the 0c.19 completion report. Takes about 20-40 min of downloads (one-time).
