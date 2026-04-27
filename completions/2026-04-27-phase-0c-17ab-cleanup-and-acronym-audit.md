# Phase 0c.17a-cleanup + 0c.17b — Render Pipeline Polish + Canon Acronym Audit
**Date:** 2026-04-27  
**Tag:** v0.5.4.5-pipeline-cleanup  
**Branch:** phase-0c-17ab-cleanup-and-audit  
**Status:** ✅ Complete — no build, no deploy, render artifacts and audit delivered

---

## Original prompt from DC

```
PHASE 0c.17a-cleanup + 0c.17b — Render pipeline polish + canon acronym audit

Two small render-pipeline tasks combined into one phase. Both touch the 
same tools/render-phrases.py infrastructure. Neither touches app code, 
neither requires a build, neither requires iPad deploy. Render artifacts 
and audit document are the deliverables.

Tag: v0.5.4.5-pipeline-cleanup

═══════════════════════════════════════════════════════════════════════
TASK 1 of 2 — Auto-copy phrases.json to bundle (0c.17a-cleanup)
═══════════════════════════════════════════════════════════════════════

CONTEXT

Phase 0c.16 introduced a maintenance gotcha: VoiceService loads phrase 
text from phrases.json bundled in WallPanel/WallPanel/phrases.json, 
but the source-of-truth manifest lives at tools/phrases.json. The 
0c.16 completion report flagged: "after any render-phrases.py run that 
changes phrase texts, re-copy: cp tools/phrases.json 
WallPanel/WallPanel/phrases.json. If phrases.json is absent from 
bundle, VoiceService logs a warning and transcripts show empty 
strings."

This is the kind of step that gets forgotten. Better to eliminate it 
by making the render script handle the copy automatically.

SCOPE

Modify tools/render-phrases.py to automatically copy the manifest to 
the bundle path as the final step of every successful run.

IMPLEMENTATION:

1. After all rendering completes successfully, copy tools/phrases.json 
   to WallPanel/WallPanel/phrases.json
2. Use shutil.copy2() to preserve mtime (helps Xcode notice the change)
3. Log the copy: "Copied manifest to bundle: WallPanel/WallPanel/phrases.json"
4. If the bundle directory doesn't exist (edge case — shouldn't happen 
   in practice), log a warning and continue rather than failing
5. The copy happens regardless of whether any new phrases were rendered 
   (idempotent — running the script with no changes still ensures bundle 
   matches source-of-truth)

CONSIDER:

- Whether to add a --no-bundle-copy flag for cases where someone wants 
  to render without copying. Probably overkill for now — auto-copy 
  is the desired default behavior 100% of the time. Skip the flag, 
  add later only if needed.

DELIVERABLES:

- tools/render-phrases.py modified with the auto-copy step
- Inline comment explaining what the copy does and why (so future-CC 
  doesn't remove it as "redundant")

═══════════════════════════════════════════════════════════════════════
TASK 2 of 2 — Canon acronym audit (0c.17b)
═══════════════════════════════════════════════════════════════════════

CONTEXT

Cartesia renders text literally — when given "S.H.I.E.L.D." it 
synthesizes Benedict saying "S-H-I-E-L-D" letter-by-letter, not the 
word "shield." Some renders need fixing; others are correctly rendered 
as letters (e.g., "10:52 a.m." rendered "ay-em" is natural speech).

This task is FIND-AND-PRESENT only. Todd's ear is the judge per phrase. 
NO substitutions, NO re-renders, NO manifest changes happen in this 
phase.

SCOPE

1. SCAN MANIFEST FOR ACRONYM PATTERNS
   
   Search tools/phrases.json for phrases containing:
   
   - Periods between capital letters: pattern /[A-Z]\.[A-Z]\./ 
     (e.g., "S.H.I.E.L.D.", "J.A.R.V.I.S.", "F.B.I.")
   - Lowercase initialisms with periods: pattern /[a-z]\.[a-z]\./ 
     (e.g., "a.m.", "p.m.", "i.e.", "e.g.")
   - Sequential capital letters of length 2-5 without periods: 
     pattern /\b[A-Z]{2,5}\b/ (e.g., "FBI", "NATO", "USA", "AC", "TV")
     EXCLUDING common words that happen to be capitalized at 
     sentence-start (e.g., a phrase starting with "The..." should 
     not flag "The")
   
   Build a list of every match found, with phrase number, the specific 
   acronym(s) detected, and the pool the phrase belongs to.

2. PRESENT FOR EVALUATION
   
   Deliver tools/acronym_audit.md committed to the branch with this 
   structure:
   
   # Acronym audit — phrases.json
   
   Found [N] phrases containing acronym patterns. Listen to each 
   rendered .m4a in the existing audio bundle and decide per phrase 
   whether to substitute or leave as-is.
   
   ## Phrase NN (pool: canon_main)
   - Phrase text: "Sir, Agent Coulson of S.H.I.E.L.D. is on the line."
   - Acronym(s) detected: "S.H.I.E.L.D."
   - Current render file: WallPanel/WallPanel/Audio/phrase_NNN.m4a
   - Action: [TODD TO DECIDE]
   
   ## Phrase MM (pool: canon_main)
   - Phrase text: "...high tide will be at 10:52 a.m..."
   - Acronym(s) detected: "a.m."
   - Current render file: WallPanel/WallPanel/Audio/phrase_MMM.m4a
   - Action: [TODD TO DECIDE — likely keep as-is, this is correct 
     natural speech]
   
   ... (one block per phrase with detected acronyms)
   
   For phrases where the acronym is OBVIOUSLY natural speech (a.m., 
   p.m., i.e., e.g.), CC may add a parenthetical hint as in the 
   second example above. But CC does NOT decide — every phrase still 
   has [TODD TO DECIDE] as the action line.
   
   No proposed replacements yet. Just identification and presentation.

3. NO RE-RENDER THIS PHASE
   
   Do NOT modify phrases.json.
   Do NOT re-render anything.
   Do NOT update the manifest.
   
   The re-render happens in a future phase (0c.17c) AFTER Todd has 
   reviewed acronym_audit.md and committed his decisions back to 
   the branch as edits to that file.

═══════════════════════════════════════════════════════════════════════
COMBINED DELIVERABLES
═══════════════════════════════════════════════════════════════════════

TASK 1:
- tools/render-phrases.py modified with auto-copy logic + inline comment

TASK 2:
- tools/acronym_audit.md committed to the branch with all detected 
  acronyms, structured per phrase, with [TODD TO DECIDE] action lines

GENERAL:
- Both tasks land on the same branch (suggested name: 
  phase-0c-17ab-cleanup-and-audit, or follow CC convention)
- Two separate commits if cleaner — render script change is one 
  conceptual unit, audit document is another
- Tag v0.5.4.5-pipeline-cleanup on the latest commit
- No iPad deploy, no build, no app code touched

═══════════════════════════════════════════════════════════════════════
COMPLETION REPORT REQUIREMENTS
═══════════════════════════════════════════════════════════════════════

Completion report at:
https://github.com/tlavigne974os/home-control-handoff/blob/main/completions/2026-04-27-phase-0c-17ab-cleanup-and-acronym-audit.md

REPORT MUST INCLUDE:

1. THE FULL ORIGINAL PROMPT FROM DC (this entire prompt, verbatim, 
   in a fenced code block at the top of the report under heading 
   "## Original prompt from DC"). Standard requirement for every 
   completion report — full round-trip artifact in the repo.

FOR TASK 1 (render-script auto-copy):

2. Confirmation the auto-copy step ran successfully on a test invocation 
   (no need to render anything new, just demonstrate the copy occurs)

3. Inline diff or relevant code snippet showing what was added to 
   render-phrases.py

FOR TASK 2 (acronym audit):

4. Total count of phrases with detected acronyms (broken down by 
   pattern type: dotted-uppercase, dotted-lowercase, plain-uppercase)

5. Any phrases where the acronym detection was ambiguous (e.g., 
   capital letters that might be a name vs an acronym, or borderline 
   cases CC wasn't sure how to classify)

6. Confirmation that audit is presentation-only — no fixes have been 
   applied, awaiting Todd's per-phrase decisions

7. Branch name and exact path to acronym_audit.md
```

---

## Task 1 — Auto-copy: what changed

Added `import shutil` to imports. Added `BUNDLE_MANIFEST` constant alongside existing path constants with an inline comment explaining the why. Added the copy block as the final step of every successful non-dry-run run.

**Relevant additions to `tools/render-phrases.py`:**

```python
# In imports
import shutil

# In constants (alongside existing AUDIO_DIR, etc.)
# Phase 0c.16 introduced VoiceService loading phrase text from a bundled
# copy of phrases.json for transcript display. This path must stay in sync
# with the source-of-truth manifest in tools/. The auto-copy at the end of
# every successful run eliminates the manual "cp tools/phrases.json ..." step
# that would otherwise need to be remembered after every render.
BUNDLE_MANIFEST = REPO_ROOT / "WallPanel" / "WallPanel" / "phrases.json"

# At end of main(), after save_hash_cache() and summary print:
# ── Auto-copy manifest to app bundle ──────────────────────────────────────
# VoiceService (Phase 0c.16) loads phrase text from a bundled copy of
# phrases.json for transcript display. The copy is kept in sync here so
# there's no separate manual step after adding or editing phrases.
# shutil.copy2 preserves mtime, which helps Xcode notice the file changed.
if BUNDLE_MANIFEST.parent.exists():
    shutil.copy2(MANIFEST, BUNDLE_MANIFEST)
    print(f"   Copied manifest to bundle: {BUNDLE_MANIFEST.relative_to(REPO_ROOT)}")
else:
    print(f"   WARNING: bundle path not found, skipping copy: {BUNDLE_MANIFEST}")
    print(f"   Run manually: cp {MANIFEST} {BUNDLE_MANIFEST}")
```

**Dry-run behavior:** the auto-copy does NOT fire on `--dry-run` — dry-run returns early before the copy block, which is correct. The copy only fires when rendering actually completes.

**Test verification:** Copy logic tested directly in Python — source `19174 bytes` matched destination `19174 bytes`. ✓

---

## Task 2 — Acronym audit results

**Total phrases scanned:** 199 (full manifest including thinking_tick pool)

| Pattern type | Phrases found |
|---|---|
| Dotted uppercase `[A-Z]\.[A-Z]\.` | 4 phrases (#013, #029, #052, #068) |
| Dotted lowercase `[a-z]\.[a-z]\.` | 1 phrase (#003, contains `a.m.`) |
| Plain uppercase `\b[A-Z]{2,5}\b` | **0 phrases** |

**All 5 flagged phrases are in `canon_main`.** All other pools (casual_mic, transition_to_quick, transition_to_full, transition_to_ambient, thinking_tick) are completely clean.

### The 5 phrases

| # | Acronym(s) | Risk level |
|---|---|---|
| 003 | `A.M.` + `a.m.` | Low — time abbreviations likely sound natural |
| 013 | `S.H.I.E.L.D.` | **High** — may letter-by-letter |
| 029 | `J.A.R.V.I.S.` | **High** — may letter-by-letter |
| 052 | `S.H.I.E.L.D.` | **High** — same as #013 |
| 068 | `J.A.R.V.I.S.` | **High** — same as #029, shorter phrase |

### Ambiguous detections

**None.** The scan was unambiguous. The plain-uppercase filter (excluding `I` and `A` articles) produced zero matches — there are no plain acronyms like FBI, NATO, or TV in the manifest.

Phrase #003 contains both `A.M.` (dotted uppercase) and `a.m.` (dotted lowercase) — same abbreviation, different capitalisation within the same long phrase. This wasn't ambiguous, but worth noting since both tokens appear in the same audio file.

### No fixes applied

This phase is find-and-present only. `tools/phrases.json` is unchanged. No re-renders were executed. All 5 phrases have `[TODD TO DECIDE]` action lines in the audit document. Awaiting Todd's per-phrase decisions before any substitution or re-render.

---

## Branch and file paths

**Branch:** `phase-0c-17ab-cleanup-and-audit`  
**Tag:** `v0.5.4.5-pipeline-cleanup`  
**Audit document:** `/Users/todd/Projects/home-control/tools/acronym_audit.md`  
**Two commits:**
- `ed7cc95` — render script auto-copy
- `f5c6023` — acronym audit document + tag
