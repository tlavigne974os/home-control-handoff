PHASE 0c.22 — Voice pipeline fixes (three surgical changes)

Implementing the three fixes from your diagnostic reports. All 
three live in AppleSpeechCoordinator.swift. Two distinct user-
visible bugs (instant-fail-on-tap and 8-second tick) addressed by 
three co-located code changes.

Task 0: Commit this prompt verbatim to BOTH the relay and the 
GitHub completion report URL before any other work.

═══════════════════════════════════════════════════════════════════
SCOPE — THREE FIXES
═══════════════════════════════════════════════════════════════════

Fix 1: defaultTaskHint = .search (one-line change)
Fix 2: Generation counter UUID for stale-callback guard
Fix 3: Remove teardownAudio() call from stale finalize() path

All three in AppleSpeechCoordinator.swift. No changes to 
VoiceCoordinator or VoiceService. No changes to system prompt. 
No new features.

═══════════════════════════════════════════════════════════════════
FIX 1 — defaultTaskHint
═══════════════════════════════════════════════════════════════════

AppleSpeechCoordinator.swift line ~75:

  newRecognizer?.defaultTaskHint = .search  // was .dictation

Drops post-speech silence detection from ~2-3s to ~0.5-1s. 
Expected impact: tick fires roughly 2-3 seconds sooner after 
user stops speaking.

═══════════════════════════════════════════════════════════════════
FIX 2 — Generation counter for stale callback guard
═══════════════════════════════════════════════════════════════════

The bug per your diagnosis: when a previous turn's recognition 
task gets cancelled, its callback may still fire on the OS 
background thread and dispatch a finalize() onto MainActor. If 
that arrives between the next turn's startListening() and 
waitForAutoFinalize(), it sets finalTranscriptTime non-nil with 
empty transcript, causing waitForAutoFinalize() to return "" 
immediately.

Implement the UUID generation counter pattern from your proposal:

Add stored property:

  private var currentSessionId: UUID = UUID()

In startListening(), at the start (before any other setup):

  let sessionId = UUID()
  currentSessionId = sessionId
  // ... existing reset of finalTranscriptTime, latestTranscript, 
  //     maxDurationTask, etc.

In the recognition task callback closure:

  recognitionTask = recognizer?.recognitionTask(with: request) { 
      [weak self] result, error in
      guard let self, self.currentSessionId == sessionId else { 
          return 
      }
      if let result {
          Task { @MainActor in
              guard self.currentSessionId == sessionId else { 
                  return 
              }
              // ... existing result handling ...
          }
      }
      if error != nil {
          Task { @MainActor in
              guard self.currentSessionId == sessionId else { 
                  return 
              }
              self.finalize()
          }
      }
  }

The sessionId is captured in the callback closure. When a new 
turn starts, currentSessionId changes. Stale callbacks compare 
their captured sessionId against currentSessionId, find a 
mismatch, return silently. No state to manage, no flags.

═══════════════════════════════════════════════════════════════════
FIX 3 — Don't call teardownAudio from stale finalize
═══════════════════════════════════════════════════════════════════

You noted in diagnosis that even with the generation counter 
guard, finalize() currently calls teardownAudio() which could 
destroy the NEXT turn's audio engine if finalize() is reached 
through a stale path that the guard doesn't catch.

The cleanest version: finalize() should be idempotent — calling 
it twice should not double-tear-down. Either:

(a) Move teardownAudio() out of finalize() entirely, into a 
    separate explicit teardown path that the pipeline orchestrator 
    calls at the right time, or
(b) Guard teardownAudio() with a flag that prevents double-call.

Option (a) is architecturally cleaner. Option (b) is a smaller 
change. Pick the one that's lower risk to ship. Document which 
you chose and why in the completion report.

═══════════════════════════════════════════════════════════════════
SIMULATOR VERIFICATION REQUIRED BEFORE SHIPPING
═══════════════════════════════════════════════════════════════════

Before installing on iPads, run the 7 simulator test cases from 
the previous prompt:

  1. Single tap, say nothing
  2. Single tap, say "hello Baxter"
  3. Single tap, say one word and stop
  4. Single tap, say a long sentence
  5. Rapid double-tap (this specifically tests fix 2)
  6. Tap while previous response is playing
  7. Network offline tap

For each test case, confirm in the completion report what you 
observed. Specifically for case 5 (rapid double-tap): the second 
tap should now either (a) be rejected by the concurrent pipeline 
guard cleanly, or (b) start a new listening session correctly. 
Either is acceptable. What's NOT acceptable is the second tap 
firing instant-failure visible-grace.

If computer-use access remains unavailable, run as much of the 
suite as you can without UI tap simulation. Cases that require 
UI taps can be deferred to device install — but document clearly 
in the completion report which cases were simulator-tested vs 
which were not.

═══════════════════════════════════════════════════════════════════
DEFERRED — DO NOT IMPLEMENT THIS PHASE
═══════════════════════════════════════════════════════════════════

You proposed Fix C — minimum tick duration to preserve audible 
feedback when the tick is cut short. Do NOT implement this in 
0c.22. The right sequence is: ship the latency fix, see how the 
tick feels at proper timing, then decide if minimum-duration is 
needed. Pre-emptive implementation of UX features for problems 
that may not exist after the underlying latency is fixed is the 
wrong order.

You also noted the per-turn recognizer initialization adds ~1-1.5s 
of overhead. Do NOT implement pre-warm in 0c.22. Same reasoning: 
ship the targeted fixes first, see how the pipeline feels, then 
decide if the pre-warm work is justified.

Both are good ideas. Both wait.

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

App repo (~/Projects/home-control):
- AppleSpeechCoordinator.swift updated (three fixes)
- Build succeeds, no new warnings
- Simulator verification of test cases (with honest reporting of 
  which were verified vs deferred)
- Install on both iPads OR confirm device unreachable per 
  standing rule
- All committed to main

REPORT MUST INCLUDE:
- Task 0 confirmation (relay URL + GitHub URL)
- Confirmation Fix 1 applied (line number)
- Confirmation Fix 2 applied (sessionId pattern, line numbers, 
  whether the existing result handler also got the guard)
- Confirmation Fix 3 applied (which option a or b, why, line 
  numbers)
- Simulator test results — which of the 7 cases ran cleanly
- Per-iPad install confirmation OR confirmed-unreachable state
- The build number this becomes
- Any architectural surprises encountered

POSTING THE COMPLETION REPORT:
Use the corrected header pattern:

  curl -sS -X POST \
    -H "X-Auth-Token: $RELAY_TOKEN" \
    -H "X-Kind: completion" \
    -H "X-Title: phase-0c-22-voice-fixes" \
    -H "X-From-Instance: CC" \
    --data-binary @report.md \
    "$RELAY_URL/upload"

Also commit to GitHub at:
completions/2026-04-29-phase-0c-22-voice-fixes.md
