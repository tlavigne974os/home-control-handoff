PHASE 0c.23 — Enable Mac Catalyst for native voice testing

Build 74 is exhibiting the same instant-fail-on-tap symptom that 
build 73 had, despite the Fix 2 generation counter. The simulator 
can't run STT (workgroup RPC timeout). iPad iteration is slow.

Mac Catalyst lets the app run natively on Todd's Mac with real 
microphone access and a usable Console.app debugging experience. 
This phase enables that build target.

Task 0: Commit this prompt verbatim to BOTH the relay and the 
GitHub completion report URL before any other work.

═══════════════════════════════════════════════════════════════════
SCOPE
═══════════════════════════════════════════════════════════════════

Enable Mac Catalyst as a build destination for the WallPanel 
app. Get build 75 running natively on Todd's Mac. Confirm voice 
mic-tap behavior on Mac so we can iterate quickly without iPad 
round-trips.

This is configuration work, not feature work. Goal: a working 
Mac build of the existing app, no UI redesign, no platform-
specific feature changes.

═══════════════════════════════════════════════════════════════════
PART 1 — ENABLE MAC CATALYST IN PROJECT
═══════════════════════════════════════════════════════════════════

1. Open WallPanel.xcodeproj in Xcode (or edit project.pbxproj 
   directly if Xcode GUI access is unavailable)

2. WallPanel target → General → Supported Destinations → add 
   "Mac (Designed for iPad)" or "Mac Catalyst" — pick whichever 
   gives full AppKit access

3. Verify the target builds for Mac Catalyst:
   xcodebuild -project WallPanel/WallPanel.xcodeproj \
              -scheme WallPanel \
              -destination 'platform=macOS,variant=Mac Catalyst' \
              build

4. If Mac Catalyst variant fails (e.g. capabilities mismatch, 
   entitlement issues), document what failed and fall back to 
   "Mac (Designed for iPad)" which has fewer requirements.

═══════════════════════════════════════════════════════════════════
PART 2 — RESOLVE PLATFORM-SPECIFIC ISSUES
═══════════════════════════════════════════════════════════════════

Likely issues to handle:

2.1 Microphone permission — Info.plist already has 
    NSMicrophoneUsageDescription. Should carry through. Verify.

2.2 SFSpeechRecognizer — supported on Mac Catalyst. Should work 
    without changes.

2.3 AVAudioEngine — supported on Mac Catalyst. May behave 
    slightly differently than iOS but should record and process 
    audio.

2.4 Picovoice/Porcupine — currently disabled (wakeWordEnabled = 
    false from 0c.21). Doesn't need to work on Mac. If the build 
    fails because Porcupine SPM doesn't support Mac Catalyst, 
    wrap the Porcupine import in #if !targetEnvironment(macCatalyst) 
    or similar to compile-out on Mac.

2.5 iCloud Shared Album photo loading — may not work on Mac the 
    same way as iOS. Acceptable to have a degraded ambient view 
    on Mac. Voice testing is the goal, not full feature parity.

2.6 HA polling, Cartesia API, Anthropic API — all networked, 
    should work identically.

2.7 Keychain — works on Mac with an entitlement. The seeded 
    API keys may need re-seeding for the Mac build. Document if 
    so.

═══════════════════════════════════════════════════════════════════
PART 3 — RUN BUILD 75 ON MAC, CAPTURE THE INSTANT-FAIL
═══════════════════════════════════════════════════════════════════

After build 75 runs on Mac:

3.1 Launch the app. Console.app open, filter on `[Baxter]`.

3.2 Tap the mic button. Note exactly what happens:
    - Does the visible-grace message fire instantly (same as 
      iPad)?
    - Or does the Mac version behave differently?

3.3 Capture the full Console output for that single mic-tap, from 
    the moment of tap through the visible-grace appearance. We 
    need to see:
    - presentationState transitions (should print via the didSet 
      you added in 0c.22)
    - startListening() being called
    - The recognizer setup steps
    - Whether STT actually starts or fails silently
    - Where the empty-transcript visible-grace path gets triggered

3.4 If the Mac version reproduces the iPad bug: capture the log 
    and post it to the relay as kind=misc with title 
    "instant-fail-mac-console-log". This becomes the diagnostic 
    artifact for the next investigation.

3.5 If the Mac version does NOT reproduce the iPad bug (mic 
    actually listens on Mac, fails on iPad): that itself is a 
    finding. Post the Mac console log AND a description of the 
    iPad behavior comparison.

═══════════════════════════════════════════════════════════════════
PART 4 — DEFERRED, NOT FOR THIS PHASE
═══════════════════════════════════════════════════════════════════

Do NOT, in this phase:

- Try to fix the instant-fail bug. The point of this phase is to 
  get a usable testing environment. Bug fix is a separate phase 
  that comes after we have logs.
- Add Mac-specific UI adjustments. The iPad UI on a Mac window 
  may look weird; that's fine.
- Try to make Porcupine work on Mac.
- Try to make iCloud Shared Album work on Mac.

The success criterion is: app launches on Mac, you can tap the 
mic, you can capture Console output for the result.

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

App repo:
- WallPanel.xcodeproj configured with Mac Catalyst destination
- Any platform-specific compile guards added (Porcupine, iCloud 
  if needed)
- Build 75 runs on Todd's Mac
- Committed to main

Diagnostic artifact:
- Console log from mic-tap on Mac, posted to relay as kind=misc

REPORT MUST INCLUDE:
- Task 0 confirmation (relay URL + GitHub URL)
- Mac Catalyst variant chosen and why
- Any platform-specific issues hit and how resolved
- Confirmation app launches on Mac
- Mic-tap behavior on Mac (does it reproduce the bug?)
- Console log URL on relay
- Build number this becomes
- Per-iPad install confirmation OR confirmed-unreachable per 
  standing rule (build 75 should still be installed on iPads 
  even though Mac is the testing path now)

POST WITH CORRECT HEADERS:
  curl -sS -X POST \
    -H "X-Auth-Token: $RELAY_TOKEN" \
    -H "X-Kind: completion" \
    -H "X-Title: phase-0c-23-mac-catalyst" \
    -H "X-From-Instance: CC" \
    --data-binary @report.md \
    "$RELAY_URL/upload"

Commit to GitHub at:
completions/2026-04-29-phase-0c-23-mac-catalyst.md
