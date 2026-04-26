# Phase 1.5b.1 — Cartesia Model Correction + Phrase 110 Audit
**Date:** 2026-04-26  
**Tag:** v0.5.4.1-cartesia-fix  
**Build:** 0.5.4 / build 63  
**Status:** ✅ Complete — deployed to iPad Pro + iPad Air

---

## Item 1 — Model switch: sonic-2 → sonic-3 ✅

`CARTESIA_MODEL` in `tools/render-phrases.py` updated from `"sonic-2"` to `"sonic-3"`.

sonic-3 confirmed working with Benedict Royal Narrator (`7cf0e2b1-8daf-4fe4-89ad-f6039398f359`) — HTTP 200 on test render before running full batch.

All 169 phrases re-rendered with `--force`.

### Re-render stats

| Metric | Value |
|--------|-------|
| Phrases rendered | 169 |
| Skipped | 0 (--force) |
| Errors | 0 |
| Render time | 188.0s |
| Data transferred | 6.6MB |

Slightly smaller payload than sonic-2 run (6.9MB → 6.6MB) — same bitrate target, likely different internal encoding efficiency. No quality regression expected; sonic-3 is the improved model.

### Subjective quality note

No A/B comparison was performed by CC (audio playback not available in this environment). Todd should A/B in the room. If sonic-3 sounds noticeably different/better, no action needed. If any phrase has odd pronunciation, use `--number NNN` to re-render individually after adjusting text in the manifest.

---

## Item 2 — Phrase 110 audit ✅

**Finding: phrase 110 was never part of the canon_main bank. The DC spec's "1-110" count was a counting error.**

Evidence:
- `phrase_110.m4a` never existed on disk at any point in the project
- `grep canon_main tools/phrases.json | wc -l` → **109**
- The original phrase list provided by Todd/DC contained exactly 109 distinct phrases
- The gap between phrase_109 (canon_main) and phrase_111 (casual_mic) is intentional — 110 is simply an unused number in the sequence

**Resolution:** No phrase added. canon_main is 109 phrases. The manifest has no gaps within pools — numbering is sparse across the full 1-170 range by design (pools occupy ranges, not a contiguous sequence).

The completion report for v0.5.4 incorrectly stated "reserved/skipped intentionally" — the more accurate statement is "DC spec said 110 but provided 109 phrase texts; the count was off by one."

---

## Files changed

| File | Change |
|------|--------|
| `tools/render-phrases.py` | `CARTESIA_MODEL = "sonic-3"` |
| `WallPanel/Audio/phrase_001–170.m4a` | All 169 re-rendered with sonic-3 |
| `WallPanel/Info.plist` | Build 63 |

---

## Next phase

**Phase 0d.2 — Write implementation.** All 8 HAController stubs throw `.notImplemented`.
