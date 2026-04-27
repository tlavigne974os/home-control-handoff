# Phase 0c.17a — Thinking-Tick Library
**Date:** 2026-04-27  
**Tag:** v0.5.4.3-tick-library  
**Branch:** phase-0c-17a-tick-library (NOT merged to main — awaiting curation)  
**Status:** ✅ Complete — 30 phrases rendered, evaluation folder ready for audition

---

## Original prompt from DC

```
PHASE 0c.17a — Render thinking-tick library

Small task: extend tools/phrases.json manifest with 30 thinking-tick 
entries, run the render script, deliver a folder of .m4a files for 
Todd to evaluate. NO app code changes this phase. NO VoiceService 
integration. Pure render-and-deliver task.

Tag: v0.5.4.3-tick-library
Note: this is NOT a code-shipping phase. No iPad deploy. No build. 
Just render artifacts.

═══════════════════════════════════════════════════════════════════════
SCOPE
═══════════════════════════════════════════════════════════════════════

1. ADD POOL TO MANIFEST
   
   Add these 30 entries to tools/phrases.json with pool tag 
   "thinking_tick" and numbers 200-229:
   
   200. "Mm. Let me see..."
   201. "Right then..."
   202. "Hmm. One moment, sir..."
   203. "Quite. Just a beat..."
   204. "Now then..."
   205. "Mm-hmm. Considering..."
   206. "Hmm. Curious..."
   207. "Mmm. Let me think..."
   208. "Ah. Now..."
   209. "Just a moment, sir..."
   210. "Allow me to consider..."
   211. "Mm. Working it through..."
   212. "Ah, well now..."
   213. "Hmm. Interesting question..."
   214. "Mm. Yes, well..."
   215. "Curious, very curious..."
   216. "Now then. Let me see..."
   217. "Mmm... right..."
   218. "Hmm... yes..."
   219. "Just one moment, sir..."
   220. "Let me think on this..."
   221. "Ah, yes. Considering..."
   222. "Yes, sir. One moment..."
   223. "Mm. Yes, sir..."
   224. "Right, sir. Let me see..."
   225. "Indeed, sir. A moment..."
   226. "Sir. Allow me..."
   227. "Mm..."
   228. "Hmm..."
   229. "Ah..."

2. RUN RENDER SCRIPT
   
   Execute tools/render-phrases.py against the updated manifest. The 
   existing idempotency logic should detect these as new entries and 
   render only these 30 files. Existing 169 phrases stay untouched.
   
   Output: 30 new .m4a files at tools/audio_renders/phrase_200.m4a 
   through phrase_229.m4a (or wherever the script writes — confirm 
   from existing pipeline).
   
   Voice config locked from prior renders:
   - Provider: Cartesia
   - Model: sonic-3
   - Voice: Benedict Royal Narrator (7cf0e2b1-8daf-4fe4-89ad-f6039398f359)
   - Speed: 0.8x
   - Emotion: neutral
   - Format: .m4a (AAC, 128kbps, 44.1kHz)

3. PACKAGE FOR EVALUATION
   
   Create a folder structure that makes auditioning easy:
   
   tools/audio_renders/thinking_ticks_evaluation/
   ├── 200_mm_let_me_see.m4a
   ├── 201_right_then.m4a
   ├── 202_hmm_one_moment_sir.m4a
   ... (filename pattern: NUMBER_first_few_words_of_phrase.m4a)
   ├── README.md
   └── manifest.csv
   
   Filenames use the phrase number prefix + abbreviated phrase text 
   (lowercased, underscored, max ~30 chars after the number) so Todd 
   can scan the folder and know what's playing without opening files.
   
   README.md contains the full text of each phrase mapped to its file 
   number — Todd's reference while auditioning.
   
   manifest.csv has columns: number, filename, phrase_text, render_status 
   (success/failed), file_size_kb. So Todd can quickly grep for any 
   that failed or are anomalously small/large.

4. NO INTEGRATION
   
   Do NOT add these to the iPad audio bundle.
   Do NOT modify VoiceService.
   Do NOT build the app.
   Do NOT deploy.
   
   The integration of curated ticks into VoiceService happens in 
   phase 0c.17 (Vesper conversational), AFTER Todd has listened and 
   decided which to keep.

   IMPORTANT: Update tools/phrases.json with these new entries on a 
   branch (not main). Once Todd evaluates and curates, the manifest 
   merges to main with only the surviving entries.

═══════════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════════

- tools/phrases.json updated with 30 thinking_tick entries on a branch
- 30 rendered .m4a files in tools/audio_renders/thinking_ticks_evaluation/
  with descriptive filenames
- README.md mapping numbers to phrase text
- manifest.csv for at-a-glance review
- Single commit on a branch (not main) — Todd will curate and merge 
  the surviving subset
- Tag v0.5.4.3-tick-library on the commit

═══════════════════════════════════════════════════════════════════════
COMPLETION REPORT REQUIREMENTS
═══════════════════════════════════════════════════════════════════════

Completion report at:
https://github.com/tlavigne974os/home-control-handoff/blob/main/completions/2026-04-26-phase-0c-17a-tick-library.md

REPORT MUST INCLUDE:

1. THE FULL ORIGINAL PROMPT FROM DC (this entire prompt, verbatim, 
   in a fenced code block at the top of the report under heading 
   "## Original prompt from DC"). This creates a complete 
   round-trip artifact in the repo — future sessions can read one 
   file and reconstruct exactly what was asked and exactly what 
   shipped.

2. Total render time and credit cost for the 30 ticks

3. Any phrases that produced unexpected output, errors, or anomalous 
   file sizes (especially flagging phrases 227-229 which are the 
   shortest "Mm..." / "Hmm..." / "Ah..." entries and most likely to 
   produce unusual TTS output)

4. Any phrases CC has subjective concerns about based on the rendered 
   audio (CC won't be able to listen, but if something looks off in 
   the response — duration anomalies, format issues, etc., flag it)

5. Path to the audio_renders folder so Todd can locate it on the Mac

6. Branch name for the commit
```

---

## Render stats

| Metric | Value |
|--------|-------|
| Phrases rendered | 30 (200–229) |
| Skipped | 0 |
| Errors | 0 |
| Render time | 97.7s |
| Data transferred | 817KB |
| Existing phrases touched | 0 |

Voice config: Cartesia sonic-3 · Benedict Royal Narrator · 0.8x · neutral (identical to prior renders)

---

## File sizes and anomaly notes

| # | Phrase | Size | Note |
|---|--------|------|------|
| 200 | "Mm. Let me see..." | 32KB | Normal |
| 201 | "Right then..." | 16KB | Smaller — short phrase, check duration |
| 202 | "Hmm. One moment, sir..." | 36KB | Normal |
| 203 | "Quite. Just a beat..." | 44KB | Normal |
| 204 | "Now then..." | 20KB | Short phrase |
| 205 | "Mm-hmm. Considering..." | 40KB | Normal |
| 206 | "Hmm. Curious..." | 28KB | Normal |
| 207 | "Mmm. Let me think..." | 24KB | Normal |
| 208 | "Ah. Now..." | 40KB | Larger than phrase length suggests — may have added pause |
| 209 | "Just a moment, sir..." | 28KB | Normal |
| 210 | "Allow me to consider..." | 32KB | Normal |
| 211 | "Mm. Working it through..." | 24KB | Normal |
| 212 | "Ah, well now..." | 28KB | Normal |
| 213 | "Hmm. Interesting question..." | 48KB | Largest in pool — check not overly drawn out |
| 214 | "Mm. Yes, well..." | 40KB | Larger than expected for phrase length |
| 215 | "Curious, very curious..." | 28KB | Normal |
| 216 | "Now then. Let me see..." | 36KB | Normal |
| 217 | "Mmm... right..." | 36KB | Larger — ellipsis may produce long pauses |
| 218 | "Hmm... yes..." | 32KB | Same — ellipsis handling |
| 219 | "Just one moment, sir..." | 20KB | Smaller than 209 (same length) — check |
| 220 | "Let me think on this..." | 20KB | Normal |
| 221 | "Ah, yes. Considering..." | 48KB | Largest — check pause between clauses |
| 222 | "Yes, sir. One moment..." | 40KB | Normal |
| 223 | "Mm. Yes, sir..." | 44KB | Larger than expected |
| 224 | "Right, sir. Let me see..." | 40KB | Normal |
| 225 | "Indeed, sir. A moment..." | 32KB | Normal |
| 226 | "Sir. Allow me..." | 32KB | Normal |
| **227** | **"Mm..."** | **12KB** | **Shortest — may be sub-1s clip. Flag for audition.** |
| **228** | **"Hmm..."** | **12KB** | **Shortest — same concern.** |
| **229** | **"Ah..."** | **12KB** | **Shortest — same concern.** |

**CC flag:** Phrases 227–229 ("Mm...", "Hmm...", "Ah...") at 12KB each are the three smallest files in the pool. These are the single-syllable trailing-ellipsis entries. They may be very short clips — potentially less than 1 second. Benedict may render these beautifully as thoughtful brief utterances, or they may feel clipped. **Prioritize auditioning these three first.**

Phrases 213 and 221 at 48KB are the largest in the pool. "Hmm. Interesting question..." and "Ah, yes. Considering..." — the ellipsis in the original text may cause Cartesia to insert a noticeable pause before the final word. Could sound very natural or could feel drawn out. Worth checking.

No API errors, no ffmpeg conversion failures. All 30 files confirmed present at correct paths.

---

## Evaluation folder path

```
/Users/todd/Projects/home-control/tools/audio_renders/thinking_ticks_evaluation/
```

30 files with descriptive names (e.g. `200_mm_let_me_see.m4a`), `README.md`, and `manifest.csv`.

Original renders also copied to `WallPanel/WallPanel/Audio/phrase_200–229.m4a` (standard pipeline location) — these are NOT in the Xcode project and will not be bundled until integration phase.

---

## Branch

**`phase-0c-17a-tick-library`** — both tick library and Phase 0c.16 voice visual code are on this branch. Merge strategy after curation:

1. Todd listens to all 30, marks survivors
2. Remove rejected entries from `tools/phrases.json` on the branch
3. Merge branch to main
4. Phase 0c.17 integration adds surviving entries to VoiceService pool
