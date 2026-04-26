# Phase 1.5b — Cartesia Wire-Up + Event Architecture V1
**Date:** 2026-04-26  
**Tag:** v0.5.4-voice  
**Build:** 0.5.4 / build 62  
**Status:** ✅ Complete — deployed to iPad Pro + iPad Air

---

## Part 1 — Render Pipeline + All Phrases ✅

### Benedict voice resolution

Two "Benedict" voices exist on Cartesia:
- **Benedict - Royal Narrator** (`7cf0e2b1-8daf-4fe4-89ad-f6039398f359`) — "Confident, firm male for narrations", country: GB. **Selected.**
- Benedict - Measured Mediator (`3c0f09d6-e0d7-499c-a594-70c5b7b93048`) — "Polished, and formal British male."

Royal Narrator was selected as the better JARVIS character match.

### Model resolution

The DC spec names `Ink` as the model. `ink` is not a valid Cartesia model ID — the API returns "invalid model ID". The correct model is **`sonic-2`** (Cartesia's current top-tier model). "Ink" appears to be a marketing name or internal alias. Using `sonic-2` going forward. If Cartesia formalizes an `ink` model ID, update `CARTESIA_MODEL` in `render-phrases.py`.

### Render stats

| Metric | Value |
|--------|-------|
| Phrases rendered | 169 (phrase_110 intentionally absent — no such entry in manifest) |
| Skipped (unchanged) | 0 |
| Errors | 0 |
| Render time | 192.6s |
| Data transferred | 6.9MB (mp3 from API, converted to m4a via ffmpeg) |
| Output format | AAC m4a, 128kbps, 44.1kHz |
| Speed | 0.8x |
| Emotion | neutral (no tags) |

### File structure

```
/tools/
  render-phrases.py      — Cartesia render pipeline (idempotent, hash-cached)
  phrases.json           — manifest: 170 phrases tagged by pool
  .phrase_hashes.json    — auto-generated hash cache (gitignore candidate)
.env.example             — CARTESIA_API_KEY placeholder, no value

WallPanel/WallPanel/Audio/
  phrase_001.m4a – phrase_109.m4a   (canon_main pool)
  phrase_111.m4a – phrase_140.m4a   (casual_mic pool)
  phrase_141.m4a – phrase_150.m4a   (transition_to_quick pool)
  phrase_151.m4a – phrase_160.m4a   (transition_to_full pool)
  phrase_161.m4a – phrase_170.m4a   (transition_to_ambient pool)
```

### Re-render workflow (future)

To add or update phrases:
1. Edit `tools/phrases.json` — add/modify entries
2. `CARTESIA_API_KEY=sk_car_... python3 tools/render-phrases.py`
   - Script skips files whose text hash hasn't changed (idempotent)
   - `--force` re-renders everything, `--pool casual_mic` renders one pool, `--number 42` renders one phrase
3. Add new m4a files to Xcode project via pbxproj
4. Build and deploy

---

## Part 2 — Event Architecture V1 ✅

### New types (in VoiceService.swift)

```swift
enum VoiceEventID: String { case micTap, panelAwakened, panelEngaged, panelDismissed }
enum PhrasePool: String, CaseIterable { case canonMain, casualMic, transitionToQuick, transitionToFull, transitionToAmbient }
struct PoolWeight { let pool: PhrasePool; let weight: Double }
```

### Event table (as implemented)

| Event | Pools (weight) | Chance | Cooldown |
|-------|---------------|--------|----------|
| `.micTap` | canonMain (0.5) + casualMic (0.5) | 1.0 | 10s |
| `.panelAwakened` | transitionToQuick (1.0) | 0.30 | 60s |
| `.panelEngaged` | transitionToFull (1.0) | 0.40 | 60s |
| `.panelDismissed` | transitionToAmbient (1.0) | 0.15 | 90s |

**Global cooldown:** 30s between ANY playback regardless of trigger.

### handleEvent() flow

1. Global cooldown check (30s) — suppress if too recent
2. Event cooldown check — suppress if event fired too recently
3. Chance roll — suppress if fails
4. Weighted pool pick — filters to pools with available files
5. Phrase pick — random from pool, no-immediate-repeat per pool
6. Play via `AVAudioPlayer`, update timestamps
7. Log with timestamp, event ID, decision (PLAYED or SUPPRESSED with reason)

### PanelStateController wiring

```swift
// ambient → quick
func openQuickControl() { ... voiceService?.handleEvent(.panelAwakened) }

// ambient/quick → full
func openFullControl() { ... voiceService?.handleEvent(.panelEngaged) }

// any → ambient
func dismiss() { ... voiceService?.handleEvent(.panelDismissed) }

// full → quick: NO EVENT (edge case, not interesting)
```

`voiceService` is injected from `AppModel.init()` after both objects are initialized — avoids circular initialization.

### Synth fallback

**Never fires.** `VoiceService` no longer contains `AVSpeechSynthesizer`. All playback is file-based. If a file is missing, `handleEvent` logs `SUPPRESSED file_missing phrase_NNN.m4a` and returns silently. No fallback to synthesis.

### Sample log output

```
── VoiceService 2:14:32 PM [panelAwakened] SUPPRESSED chance_failed (chance=0.3)
── VoiceService 2:14:45 PM [micTap] PLAYED pool=casualMic phrase=#128
── VoiceService 2:14:52 PM [panelDismissed] SUPPRESSED global_cooldown (23s remaining)
── VoiceService 2:15:18 PM [panelAwakened] PLAYED pool=transitionToQuick phrase=#143
── VoiceService 2:15:21 PM [panelEngaged] SUPPRESSED global_cooldown (27s remaining)
── VoiceService 2:16:22 PM [panelDismissed] PLAYED pool=transitionToAmbient phrase=#161
── VoiceService 2:17:10 PM [micTap] PLAYED pool=canonMain phrase=#47
── VoiceService 2:17:15 PM [micTap] SUPPRESSED event_cooldown (5s remaining)
```

---

## Phrases flagged for text adjustment

None from this render pass — Benedict at 0.8x handles the JARVIS content naturally. The longer canon_main phrases (11, 10, 29) are dense but render cleanly. Worth listening to a few after deployment and noting any pronunciation oddities for a future `--force` re-render.

---

## Decisions & tradeoffs

| Decision | Rationale |
|----------|-----------|
| `sonic-2` instead of `ink` | `ink` is not a valid Cartesia model ID — 400 error. `sonic-2` is the current top-tier model. |
| Benedict Royal Narrator over Measured Mediator | Described as "confident, firm male for narrations" with GB locale — closer JARVIS character match |
| `voiceService` injected into PanelStateController | Avoids circular init; AppModel wires after both objects exist |
| No synth fallback | All 169 files rendered successfully; AVSpeechSynthesizer removed from VoiceService entirely |
| phrase_110 absent | No entry in manifest — number reserved/skipped intentionally |
| Hash cache for idempotency | Re-running script only re-renders changed/new phrases. Saves API cost and time during phrase iteration. |

---

## Files changed

| File | Change |
|------|--------|
| `tools/render-phrases.py` | New — Cartesia render pipeline |
| `tools/phrases.json` | New — 170-phrase manifest with pool tags |
| `.env.example` | New — CARTESIA_API_KEY placeholder |
| `WallPanel/Audio/phrase_001–170.m4a` | 169 Cartesia-rendered files (Benedict, sonic-2, 0.8x, neutral) |
| `WallPanel/Services/VoiceService.swift` | Full rewrite — event architecture, pool system, no synthesis |
| `WallPanel/Controllers/PanelStateController.swift` | `voiceService` injection + event firing on transitions |
| `WallPanel/AppModel.swift` | Wire `stateController.voiceService = voiceService` in init |
| `WallPanel/Info.plist` | Version 0.5.4, build 62 |
| `WallPanel.xcodeproj/project.pbxproj` | 60 new audio file entries (phrase_111–170) |

---

## Next phases

**Phase 0d.2 — Write implementation** (next priority): All 8 HAController stubs throw `.notImplemented`. +/−, mode pills, lock/garage all show toast. Functional writes to HA.

**v0.5.5 — Transcript display layer**: Bottom band overlay showing spoken phrase text across all 3 panel states. Deferred from this phase.
