# Phase 0c.20 — Baxter System Prompt v2 + Character Versioning

**Branch:** `main` (committed directly — configuration, not a feature branch concern)  
**Build:** 71  
**Date:** 2026-04-28  
**Status:** ✅ Complete

---

## Task 0 — Prompt Verbatim (per spec)

```
PHASE 0c.20 — Baxter system prompt v2 + character versioning infrastructure

Three character calibrations, plus the versioning discipline that makes 
Baxter's prompt durable: preserved file versions, mirrored to public 
handoff repo, sync script for future iterations.

The three character changes are specified in the attached file:
baxter-system-prompt-v2-edits.md

Task 0: Commit this prompt verbatim to the completion report URL before 
any other work.

═══════════════════════════════════════════════════════════════════
SCOPE
═══════════════════════════════════════════════════════════════════

PART A — CREATE BAXTER SYSTEM PROMPT v2

Step 1: Read the full content of v1 at:
WallPanel/WallPanel/Resources/baxter-system-prompt-v1.md

This is the source of truth. v2 is a copy with three targeted section 
changes applied per the attached edits file. All other content (identity, 
address conventions, acknowledgment phrases, examples, household members, 
etc.) carries forward unchanged.

Step 2: Read the attached edits file (baxter-system-prompt-v2-edits.md) 
which Todd is providing alongside this prompt. It contains three edits:

- EDIT 1: REPLACE the NAME HANDLING section
- EDIT 2: REPLACE the LENGTH (or BREVITY) section  
- EDIT 3: INSERT a NEW section CHARACTER CALIBRATION — RESTRAINT, 
  placed immediately after VOICE CHARACTER

Step 3: Create v2 as a new file at:
WallPanel/WallPanel/Resources/baxter-system-prompt-v2.md

DO NOT DELETE v1. Both files exist alongside each other permanently. 
Versioning discipline preserves character history.

PART B — UPDATE CLAUDEVOICEASSISTANT TO LOAD v2

In WallPanel/WallPanel/Services/Voice/ClaudeVoiceAssistant.swift, find 
where the system prompt is loaded (likely in init() via Bundle.main.path 
or similar) and update the resource name from "baxter-system-prompt-v1" 
to "baxter-system-prompt-v2".

Both files remain in the bundle; only the loaded reference changes.

PART C — MIRROR CHARACTER FILES TO PUBLIC HANDOFF REPO

Add a new directory in the public handoff repo at 
github.com/tlavigne974os/home-control-handoff:

baxter-character/
├── README.md (NEW)
├── baxter-system-prompt-v1.md (mirror of app repo v1)
└── baxter-system-prompt-v2.md (mirror of app repo v2)

Write the README.md fresh, explaining:
- The system prompt is the soul of Baxter
- The canonical files live in the private app repo at 
  ~/Projects/home-control/WallPanel/WallPanel/Resources/
- These mirrors exist for: belt-and-suspenders against deletion, 
  visibility for future-Claude sessions without private repo access, 
  and character archeology showing how Baxter evolved over time
- Versioning discipline: files are named baxter-system-prompt-vN.md; 
  bump to vN+1 for substantive changes; never replace; the app loads 
  only the latest
- The sync script at tools/sync-character.sh in the app repo handles 
  mirroring

PART D — CREATE SYNC SCRIPT IN APP REPO

New file at tools/sync-character.sh in the app repo. The script should:
- Be a bash script with set -e
- Resolve the app repo path from its own location (the script lives in 
  app_repo/tools/)
- Assume the public handoff repo is checked out as a sibling at 
  ~/Projects/home-control-handoff
- Error clearly if the handoff repo is not found
- Copy all baxter-system-prompt-v*.md files from 
  WallPanel/WallPanel/Resources/ to the handoff repo's baxter-character/ 
  directory (creating the directory if needed)
- cd into the handoff repo, git add baxter-character/
- If there are staged changes, commit with message "Mirror Baxter 
  character files" and push
- If no changes, print "No character changes to mirror." and exit cleanly
- chmod +x the script after creating it

PART E — RUN THE SYNC ONCE TO ESTABLISH INITIAL MIRROR

After completing parts A through D, run tools/sync-character.sh once. 
Confirm in the completion report that v1 and v2 files now exist in the 
public handoff repo's baxter-character/ directory with the README.

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

App repo (private):
- baxter-system-prompt-v2.md created with three section changes applied
- baxter-system-prompt-v1.md preserved unchanged (verify byte-for-byte)
- ClaudeVoiceAssistant.swift updated to load v2
- tools/sync-character.sh created and executable
- All committed to main directly (this is configuration, not a feature 
  branch concern)
- Build cleanly, no new warnings

Public handoff repo:
- baxter-character/ directory with README.md and both v1 and v2 files
- Committed and pushed to main

REPORT MUST INCLUDE:
- Task 0 confirmation
- Verification that v1 is unchanged byte-for-byte from before this phase
- The exact section names that changed in v2
- Confirmation ClaudeVoiceAssistant now loads v2 (file path / line 
  number reference)
- Confirmation of sync script execution and resulting mirror state
- Any architectural surprises encountered
- Note for future workflow: character iteration loop is now (1) edit 
  the latest baxter-system-prompt-vN.md or create vN+1, (2) build and 
  test in lived use, (3) when happy, run tools/sync-character.sh, 
  (4) both repos updated

COMPLETION REPORT AT:
https://github.com/tlavigne974os/home-control-handoff/blob/main/completions/2026-04-28-phase-0c-20-system-prompt-v2.md
```

---

## Implementation

---

## Bonus — Telemetry Unification

Not in the original spec but done in the same session per user request. All `print()` statements across the voice pipeline were changed from the `──` prefix style to a single, typeable `[Baxter]` prefix.

**Files updated:** VoiceCoordinator, VoiceService, ClaudeVoiceAssistant, AppleSpeechCoordinator, WakeWordDetector, AppModel (26 print statements total).

**How to use:** In Console.app on Mac, filter by process `WallPanel` and search `[Baxter]`. See the full voice pipeline timeline in one place.

**Sample timeline for a successful conversation:**
```
[Baxter] system prompt loaded (2847 chars)    ← app launch, v2 loaded
[Baxter] wake word started (sensitivity: 0.5) ← when Porcupine linked
[Baxter] listening started                    ← STT engaged
[Baxter] stop → transcript = "Hey, tell me a joke"
[Baxter] thinking tick playing (duration: 2.40s)
[Baxter] LLM first token received             ← Claude API latency ends here
[Baxter] first Cartesia audio chunk received  ← TTS pipeline latency ends here
[Baxter] playing raw audio ...                ← audio starts
```

**Thinking tick discovery:** `phrase_200-229.m4a` (the thinking tick pool) do not exist in the bundle. You'll see `[Baxter] pool thinkingTick: 0/30 files found` at launch — ticks are silently skipped. Adding audio files to that range will light them up without any code change. The transcript (`processingUserUtterance` state) does show correctly even without the tick audio.

---

## Files Written

| File | Change |
|------|--------|
| `WallPanel/WallPanel/Resources/baxter-system-prompt-v1.md` | **NEW** — canonical copy of v1 (character history preserved) |
| `WallPanel/WallPanel/Resources/baxter-system-prompt-v2.md` | **NEW** — v1 + three section edits |
| `WallPanel/WallPanel/Services/Voice/ClaudeVoiceAssistant.swift` | load `baxter-system-prompt-v2` instead of `woadhouse-system-prompt-v1` |
| `WallPanel/WallPanel.xcodeproj/project.pbxproj` | Both new .md files registered in PBXFileReference, PBXBuildFile, PBXGroup, Resources build phase |
| `WallPanel/WallPanel/Info.plist` | Build 70 → 71 |
| `tools/sync-character.sh` | **NEW** — copies Resources/baxter-system-prompt-v*.md to handoff repo |
| `home-control-handoff/baxter-character/README.md` | **NEW** |
| `home-control-handoff/baxter-character/baxter-system-prompt-v1.md` | **NEW** — mirror |
| `home-control-handoff/baxter-character/baxter-system-prompt-v2.md` | **NEW** — mirror |

---

## v1 Preservation

`woadhouse-system-prompt-v1.md` (original at project root): **byte-for-byte unchanged** — same content, same path, still referenced in bundle.

`baxter-system-prompt-v1.md` (new canonical copy in Resources/): identical content as v1, renamed to the baxter-vN naming scheme.

sha256 of both: identical.

---

## v2 — Sections Changed

Three sections changed; all other content carried forward verbatim:

| Section | Change |
|---------|--------|
| `## About your name` → `NAME HANDLING` | REPLACE — Baxter backronym replacing Woadhouse; self-aware tag variants |
| `## Length` → `LENGTH` | REPLACE — expanded to allow depth when explicitly invited (stories, explanations); brevity rule scoped to command-and-response |
| *(new)* `CHARACTER CALIBRATION — RESTRAINT` | INSERT after `## Voice character` — wit restraint, vocabulary discipline, alternation rule |

---

## ClaudeVoiceAssistant — Load Point

**File:** `WallPanel/WallPanel/Services/Voice/ClaudeVoiceAssistant.swift`  
**Method:** `loadSystemPrompt()` (line ~92)  
**Change:** `forResource: "woadhouse-system-prompt-v1"` → `forResource: "baxter-system-prompt-v2"`

If v2 is missing from bundle, falls through to the existing hardcoded fallback — app never crashes on missing prompt.

---

## Sync Script

**File:** `tools/sync-character.sh`  
**Run:** `cd ~/Projects/home-control && bash tools/sync-character.sh`  
**What it does:** copies `WallPanel/WallPanel/Resources/baxter-system-prompt-v*.md` to the handoff repo's `baxter-character/`, commits "Mirror Baxter character files", and pushes. Idempotent — if nothing changed, exits cleanly without a commit.

---

## Sync Execution Result

Initial run after setup:

```
Syncing Baxter character files to handoff repo...
  → baxter-system-prompt-v1.md
  → baxter-system-prompt-v2.md
Committed and pushed: Mirror Baxter character files
Done.
```

Files now in public handoff repo:  
`https://github.com/tlavigne974os/home-control-handoff/tree/main/baxter-character/`

---

## Future Character Iteration Workflow

1. Edit `WallPanel/WallPanel/Resources/baxter-system-prompt-vN.md` **or** create `baxter-system-prompt-v(N+1).md`  
2. Update `ClaudeVoiceAssistant.loadSystemPrompt()` to reference the new version name  
3. Build and test on device in lived use — evaluate whether the changes feel right  
4. When satisfied: `cd ~/Projects/home-control && bash tools/sync-character.sh`  
5. Both repos updated; character history preserved

**Never replace** an existing version file. Bump N only when you're confident a revision is an improvement. The old prompt stays as archaeological record.

---

## Architectural Surprises

**Resources subdirectory in bundle:** iOS flattens all bundle resources — `baxter-system-prompt-v2.md` in `WallPanel/WallPanel/Resources/` is accessed via `Bundle.main.url(forResource: "baxter-system-prompt-v2", withExtension: "md")` the same as a file at the root. The physical folder structure in the project is for developer organization only; Xcode copies the file flat into the bundle. No path prefix needed in the Swift load call.

**Original `woadhouse-system-prompt-v1.md` kept in place:** It's still referenced in pbxproj and still copies into the bundle. ClaudeVoiceAssistant no longer loads it (loads v2 now), but it remains in the bundle as belt-and-suspenders. It can be removed from the target in a future cleanup phase once the v1/v2 naming scheme is established.

---

## Verification Checklist

- [x] Task 0 — prompt committed verbatim before any other work
- [x] `baxter-system-prompt-v1.md` in Resources — identical content to `woadhouse-system-prompt-v1.md` (sha256 verified)
- [x] `baxter-system-prompt-v2.md` in Resources — three edits applied, all other content verbatim
- [x] `ClaudeVoiceAssistant.swift` loads `baxter-system-prompt-v2`
- [x] `tools/sync-character.sh` created and executable
- [x] Build 71 — BUILD SUCCEEDED (zero errors)
- [x] Committed to main directly
- [x] Sync run — v1 and v2 mirrored to handoff repo `baxter-character/`
- [x] `baxter-character/README.md` explains versioning discipline
- [ ] Device: install build 71 (iPad was unavailable at session end — build 70 is currently running)
- [ ] Device: Baxter responds to "What's your name?" with new NAME HANDLING section content
- [ ] Device: "Tell me a story" → honored at full length (not truncated)
- [ ] Device: second witty response is plain (CHARACTER CALIBRATION verified in lived use)
- [ ] Device: Console shows `[Baxter]` prefix on all telemetry entries (search `[Baxter]` in Console.app)
- [ ] Note: `[Baxter] pool thinkingTick: 0/30 files found` at launch = expected (phrase_200-229.m4a not in bundle)
