# Phase 0c.16-revise — Voice transcription redesign
**Date:** 2026-04-27  
**Tag:** v0.5.5-voice-revise  
**Branch:** phase-0c-16-revise  
**Status:** 🔄 In progress

---

## Original prompt from DC

```
PHASE 0c.16-revise — Voice transcription redesign (full rollback of fixup)

Phase 0c.16-fixup is in flight. STOP that work immediately. Rollback 
all uncommitted changes from the fixup phase and start fresh with this 
revised design direction. The fixup direction is being replaced 
wholesale — we're not iterating on it, we're redesigning the 
transcript surface.

If the fixup phase has already committed any work to the branch, 
revert those commits cleanly so main remains at v0.5.4 as the working 
baseline. If the fixup work is on a feature branch, abandon that 
branch.

Tag: v0.5.5-voice-revise
Branch: phase-0c-16-revise (fresh, off main at v0.5.4)

═══════════════════════════════════════════════════════════════════════
TASK 0 — FIRST ACTION: COMMIT THIS PROMPT TO COMPLETION REPORT URL
═══════════════════════════════════════════════════════════════════════

Standing requirement. Before any other work, create the completion 
report file at:

  completions/2026-04-27-phase-0c-16-revise.md

with the heading "## Original prompt from DC" followed by THIS ENTIRE 
PROMPT verbatim in a fenced code block. Commit and push that file as 
the first commit of this phase. The rest of the report gets appended 
as work progresses.

═══════════════════════════════════════════════════════════════════════
TASK 1 — Rollback fixup work
═══════════════════════════════════════════════════════════════════════

The 0c.16-fixup phase was attempting to fix Mode A/Mode B leak, add 
chrome to the transcript band, and fix positioning. None of that work 
applies to the new design direction. Roll it back.

EXECUTE:

1. Check current git state — what branch is CC on, what commits exist 
   on the fixup branch, what's in working tree
2. If on a fixup feature branch with commits: report the commit hashes, 
   then check out main and abandon the fixup branch (don't delete yet 
   — keep available for reference until 0c.16-revise ships clean)
3. If uncommitted changes exist: stash or discard depending on how 
   complete they are. Report what was being modified.
4. Verify main is at v0.5.4 (commit 56bcfed per the verify-consolidation 
   report) — that's the baseline this phase builds from
5. Create new branch: phase-0c-16-revise off v0.5.4

═══════════════════════════════════════════════════════════════════════
DESIGN INTENT — "She's writing you a note"
═══════════════════════════════════════════════════════════════════════

Voice transcription is being redesigned around a single coherent 
metaphor: Woadhouse leaves you handwritten notes. Not text on a screen, 
not UI chrome, not system messages. Notes.

The Hearth design system was built on "the future has paper" — warm 
tones, physical materials, the imperfection of real things. Transcript 
inherits that language directly. When Woadhouse speaks and her words 
appear visually, they appear as if she wrote them down on warm paper 
and left them on the bottom of the screen for you to see.

This is a substantial revision of the previous approach. Old approach 
used Instrument Serif (typeset, formal) with two-mode visual treatment 
(modal overlay in Mode A, discreet band in Mode B). New approach uses 
Caveat (handwritten) with a unified bottom-band treatment across all 
three panel states.

═══════════════════════════════════════════════════════════════════════
TASK 2 — Build the unified transcript band
═══════════════════════════════════════════════════════════════════════

A. NEW UNIFIED COMPONENT

Replace BOTH AmbientTranscriptBand (Mode B) AND VoiceModalOverlay's 
transcript element (Mode A) with a single TranscriptBand component.

The modal overlay (VoiceForm + surface dim for mic-tap) STAYS for Mode A. 
Only the transcript text rendering changes — instead of appearing inside 
the modal overlay, the transcript appears in the unified bottom band. 
This means during a mic tap:

- VoiceForm appears centered, surface dims (existing Mode A treatment)
- Transcript band appears at bottom (new unified treatment, same 
  position as Mode B)

The voice form gives the modal moment; the band gives the written note. 
They coexist during Mode A.

B. POSITIONING

The TranscriptBand sits at the absolute bottom of the screen at all 
times during voice playback, regardless of panel state.

- AMBIENT MODE: band covers the status bar (status bar is at bottom 
  in ambient — it's hidden by the band during voice playback)
- QUICK CONTROL: band sits at bottom of screen, below the quick 
  control body (status bar is at top in engaged states, doesn't 
  conflict)
- FULL CONTROL: band covers the bottom action tile row (Unlock Front 
  Door / Open Garage / Set Away / Awake) — those tiles are 
  temporarily not tappable during voice playback. Tap-to-dismiss 
  resolves this immediately.

C. SIZING

Band height matches the status bar height exactly (HearthStatusBar.height). 
This creates visual coherence — the same vertical real estate that 
normally shows time/weather is now showing Woadhouse's note. Same 
"location" in the user's spatial memory.

Band width: full screen width, edge to edge.

D. TYPOGRAPHY

- Font: Caveat (handwritten)
- Color: hearthCream (single color, no variation)
- Size: large enough to fill the band comfortably and remain glanceable 
  at typical viewing distance — start with 36pt and adjust during 
  build if needed for legibility/scale fit
- Centered horizontally and vertically within the band
- No italic markup, no color variation, no honorific highlighting, no 
  number highlighting — uniform handwritten text

This explicitly retires the prior typography rules (italic for "sir", 
hearthAmberDim for numbers). Caveat in hearthCream only. The 
handwritten character carries its own warmth and emphasis — the rules 
were solving a problem that doesn't exist when the text is 
already personal.

E. PAPER TEXTURE / MATERIAL FEEL

The band background is NOT pure black. It's a warm-dark surface that 
suggests paper-in-low-light rather than screen-glow.

Implementation:
- Base color: warm-dark, slightly amber-tinted. Suggested: blend of 
  hearthBg (or equivalent darkest project color) with a 5-10% mix 
  toward hearthAmberDim. The result should read as "warm dark" not 
  "neutral dark"
- Opacity: 75-85% — enough that underlying content is dimly visible 
  through the band, suggesting the band sits ON TOP of content rather 
  than blocking it
- Subtle texture: low-contrast noise overlay, ~2% intensity — gives 
  the surface material variation. Not a visible "paper texture" — 
  just enough that the surface isn't flat
- Optional thin top border: 1pt at hearthAmberDim 30% opacity — 
  signals the band as a deliberate "object" rather than a transparency 
  layer. Try with and without; pick what feels more "noted" vs "system"

F. SLIGHT ROTATION

When the band appears, apply a subtle rotation: between -0.5° and 
+0.5°, randomly chosen per appearance. Just enough to register as 
"physical" rather than "perfectly aligned UI."

This is the most important detail for the "she wrote me a note" feel. 
Pixel-perfect alignment is the giveaway of digital UI. A real note 
sits where it was placed, slightly off-square. The band needs that 
character.

The rotation is set on appearance and held for the duration — it does 
NOT animate or change while the band is visible. Each new appearance 
gets a fresh random angle within the range.

G. ANIMATION

- Appear: standard fade-in over 250ms ease-out as audio starts. Full 
  text visible immediately (no per-character or per-word reveal — 
  that's gimmicky and fights audio timing)
- Hold: full audio duration plus 1.5s grace period after audio ends
- Disappear: fade-out over 400ms ease-in
- Rotation set on appear, doesn't change during visibility

H. TAP-TO-DISMISS

User tap anywhere on the band:
- Audio stops immediately (audioPlayer.stop())
- Band fades out
- Underlying content (status bar in ambient, action tiles in full 
  control) reappears

Critical for full control mode where the band covers interactive 
controls. Universal gesture across all three states.

═══════════════════════════════════════════════════════════════════════
TASK 3 — Wire updated VoiceService and views
═══════════════════════════════════════════════════════════════════════

A. VoicePresentationState stays as currently defined:
   - .idle
   - .ambientSpeaking(phrase: String, source: VoiceEventID)
   - .modalSpeaking(phrase: String)
   
   Both .ambientSpeaking and .modalSpeaking trigger the unified 
   TranscriptBand to show. The difference between modes is whether 
   VoiceForm + surface dim ALSO appears (Mode A only).

B. VoiceModalOverlay is now responsible only for the modal-mode 
   visuals (VoiceForm + surface dim). Transcript rendering moves out 
   of the overlay into the unified TranscriptBand.

C. AmbientTranscriptBand is removed. Replaced by TranscriptBand.

D. RootView wiring:
   - TranscriptBand attached at root ZStack, always-on observer of 
     presentationState
   - VoiceModalOverlay attached at root ZStack, observes 
     presentationState and shows ONLY when .modalSpeaking
   - Z-order: VoiceModalOverlay (highest, covers everything) > 
     TranscriptBand (above panel content but below modal overlay) > 
     panel content

   When .modalSpeaking: both modal overlay and transcript band visible 
   When .ambientSpeaking: only transcript band visible
   When .idle: neither visible

═══════════════════════════════════════════════════════════════════════
TASK 4 — FORMAL VERIFICATION ON DEVICE — 11 CASES
═══════════════════════════════════════════════════════════════════════

Build, deploy to BOTH iPads (Pro and Air), then formally execute these 
verification cases. Document PASS / FAIL / NOTES per case.

ORIGINAL 9 CASES (revised for new design):

1. Tap mic in ambient → modal overlay appears (VoiceForm + surface 
   dim), transcript band appears at bottom covering status bar with 
   handwritten phrase, audio plays, both fade out on completion

2. Tap mic in full control → same modal overlay, transcript band at 
   bottom covering action tile row, audio plays

3. Walk up to panel (ambient → quick) → transcript band appears at 
   bottom of screen below quick control body, NO modal overlay, no 
   surface dim

4. Tap into full → transcript band appears covering action tiles, no 
   modal overlay

5. Tap mic during ambient phrase playing → existing band content 
   transitions to new phrase, modal overlay appears for mic tap

6. Tap modal overlay during playback → audio stops, both band and 
   overlay fade out

7. Status bar visibility: in ambient, status bar IS COVERED during 
   voice playback (this is the new design, intentional). In quick 
   control and full control, status bar at top remains visible 
   throughout

8. Caveat font renders correctly at chosen size, all phrases readable 
   at panel viewing distance

9. No layout shift when band appears/disappears — overlay-style 
   positioning, doesn't reflow content

NEW VERIFICATION CASES:

10. Tap-to-dismiss: tap the transcript band during playback in any 
    state → audio stops, band fades out, underlying content (status 
    bar in ambient, action tiles in full control) reappears 
    immediately and is interactive

11. "Note" feel: visual inspection. Does the band feel like a 
    handwritten note rather than a UI element? Specifically:
    - Slight rotation visible and consistent on each appearance (with 
      different rotation per appearance, within -0.5° to +0.5°)
    - Background reads as "warm-dark" not "neutral-dark"
    - Caveat text renders cleanly at chosen size
    
    Document subjective impression in completion report. Todd will 
    do final aesthetic judgment in lived use, but CC's first 
    impression on device is useful data.

CAPTURE EVIDENCE

For each case, capture screenshot or screen recording. Save paths to 
referenced in completion report.

═══════════════════════════════════════════════════════════════════════
TASK 5 — Update completion report with results
═══════════════════════════════════════════════════════════════════════

After Tasks 1-4 complete, append to the completion report:

1. TASK 1 RESULTS: rollback details, what was abandoned from fixup, 
   confirmation main is at v0.5.4 baseline, new branch name

2. TASK 2 RESULTS: 
   - TranscriptBand component implementation
   - Sizing decisions (final font size after testing)
   - Paper texture decisions (with/without thin border, etc.)
   - Rotation implementation details
   
3. TASK 3 RESULTS: VoiceService wiring changes, removed components 
   (AmbientTranscriptBand), unified band integration

4. TASK 4 RESULTS: all 11 cases with PASS/FAIL/NOTES, screenshot 
   paths, subjective "note feel" impression

5. ANY OTHER FINDINGS: bugs discovered during testing, subjective 
   impressions, things future-Todd should know

6. BUILD AND DEPLOY DETAILS: build number bump, install confirmation 
   on both iPads

7. BRANCH AND TAG: branch name, commit hashes, tag 
   v0.5.5-voice-revise applied to merged commit on main

═══════════════════════════════════════════════════════════════════════
SCOPE BOUNDARIES
═══════════════════════════════════════════════════════════════════════

OUT OF SCOPE for this phase:

- Status-bar takeover for engaged states (the band approach is 
  unified now — no special status bar handling beyond ambient mode 
  coverage)
- Phase 2 listening state work
- Streaming pipeline work
- Write-in animation (deliberately rejected — feels gimmicky)
- Per-character or per-word reveal animations
- Multiple "paper" visual styles (one consistent treatment)

═══════════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════════

- Task 0: completion report file with original DC prompt committed 
  first
- Task 1: rollback complete, main verified at v0.5.4, fresh branch 
  off main
- Tasks 2-3: code changes committed to branch
- Task 4: all 11 verification cases evaluated formally with evidence
- Task 5: completion report appended with results
- Branch merged to main after verification passes
- Tag v0.5.5-voice-revise applied to merged commit
- Both iPads (Pro and Air) running the redesigned build
- Old AmbientTranscriptBand component file removed (not just unused)
```
