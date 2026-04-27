# Phase 1.5b.2 — Mic Tap Exempt from Global Cooldown
**Date:** 2026-04-26  
**Tag:** v0.5.4.2-mic-priority  
**Build:** 0.5.4 / build 64  
**Status:** ✅ Complete — deployed to iPad Pro + iPad Air

---

## The Fix

`VoiceService.handleEvent()` restructured into two explicit branches:

### User-initiated path (.micTap)

```
1. Per-event cooldown check (10s) — suppress if double-tap
2. Pick pool (weighted random)
3. Pick phrase (no-immediate-repeat)
4. audioPlayer?.stop()          ← interrupts any currently-playing audio
5. Play new phrase
6. Update event.lastFiredAt, globalLastFiredAt, pool.lastPlayed
```

- **Global cooldown bypassed** — mic is always responsive to user
- **Interrupts audio** — user command supersedes ambient chatter
- **Resets globalLastFiredAt** — ambient events must wait 30s after mic fires

### Panel-initiated path (.panelAwakened, .panelEngaged, .panelDismissed)

```
1. Global cooldown check (30s) → SUPPRESSED("global_cooldown") if too recent
2. Event cooldown check → SUPPRESSED("event_cooldown") if too recent  
3. Chance roll → SUPPRESSED("chance_failed") if fails
4. Audio busy check → SUPPRESSED("audio_busy") if audio is playing
5. Pick pool, pick phrase, play
6. Update timestamps
```

- **Does not interrupt audio** — only the user gets to interrupt
- New suppression reason `"audio_busy"` added to log output

---

## Manual test cases (expected behavior)

| Test | Expected log |
|------|-------------|
| Tap mic → phrase plays | `[micTap] PLAYED pool=... phrase=#N` |
| Tap mic again within 10s | `[micTap] SUPPRESSED event_cooldown (Xs remaining)` |
| Tap mic again after 10s | `[micTap] PLAYED ...` (second phrase) |
| Panel transition within 30s of mic tap | `[panelAwakened] SUPPRESSED global_cooldown (Xs remaining)` |
| Tap mic during panel-playing phrase | `[micTap] PLAYED ...` (interrupts panel phrase) |
| Panel transition during mic-playing phrase | `[panelDismissed] SUPPRESSED audio_busy` |

---

## Edge cases noted

**micTap cooldown is 10s, not 0.** This is intentional — if someone double-taps within a fraction of a second (accidental), the second tap is suppressed. This also means rapid sequential mic taps wait 10s between plays. If this feels too long in practice, lower `.micTap` cooldown in the event table (it's a config, not hardcoded).

**`audioPlayer?.isPlaying`** is checked for panel events but not mic. For mic, we `stop()` before checking — the interrupt behavior is unconditional.

**`makePlayer()` helper** extracted to deduplicate the URL resolution + AVAudioPlayer construction that was inlined twice (once per branch).

---

## Files changed

| File | Change |
|------|--------|
| `WallPanel/Services/VoiceService.swift` | `handleEvent()` branched on user- vs panel-initiated; `makePlayer()` helper added; `audio_busy` suppression reason for panel events |
| `WallPanel/Info.plist` | Build 64 |

---

## Next phase

**Phase 0d.2 — Write implementation.** All 8 HAController stubs throw `.notImplemented`.
