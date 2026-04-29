PHASE 0c.24 — Fix Mac Catalyst microphone permission

Build 75 enabled Mac Catalyst but the app crashes on mic-tap due 
to missing TCC microphone permission. This phase resolves that so 
Todd can use Mac as the voice debugging environment.

Task 0: Commit this prompt verbatim to BOTH the relay and the 
GitHub completion report URL before any other work.

═══════════════════════════════════════════════════════════════════
WHY THIS MATTERS
═══════════════════════════════════════════════════════════════════

The iPad voice instant-fail bug is unresolved. iPad iteration is 
slow (build → install → console → analyze). Simulator can't run 
STT. Mac Catalyst is the only fast iteration loop available — but 
build 75 crashes at engine.start() because the app doesn't have 
TCC mic permission on Todd's Mac.

═══════════════════════════════════════════════════════════════════
TASK 1 — RUN FROM XCODE DEBUGGER WITH MAC CATALYST DESTINATION
═══════════════════════════════════════════════════════════════════

Per your prior diagnosis (Option A), launching the app from 
Xcode with the Mac Catalyst destination registers it with TCC 
properly and triggers the permission dialog the first time the 
mic is accessed.

1.1 Open Xcode (or use xcodebuild)

1.2 Select the WallPanel scheme with destination 
    "platform=macOS,variant=Mac Catalyst"

1.3 Run the app. The app should launch on Mac.

1.4 IMPORTANT — Todd needs to be present to grant mic permission:
    
    Tell Todd in a chat reply when you're at this point: "Mac 
    Catalyst app is launching. When you tap the mic for the 
    first time, macOS will show a permission dialog asking to 
    allow microphone access for Home Control. Click Allow."
    
    Then proceed to tap the mic in the running app.

1.5 If the permission dialog appears and Todd grants:
    - Verify in System Settings > Privacy > Microphone that 
      Home Control now appears with toggle ON
    - Tap the mic again to confirm STT actually starts (no 
      SIGABRT)
    - Capture Console output from the mic tap

1.6 If the permission dialog does NOT appear (the most likely 
    failure mode — TCC has subtle rules about which app launches 
    trigger the dialog):
    
    Try the alternate paths in order:
    
    a. Build, then double-click the .app bundle in Finder to 
       launch it (this often triggers the dialog more reliably 
       than Xcode debugger)
    
    b. Use the `open` command from terminal:
       open ~/Library/Developer/Xcode/DerivedData/<derived-data-path>/Build/Products/Debug-maccatalyst/Home\ Control.app
    
    c. If still no dialog, manually add the app to System 
       Settings > Privacy > Microphone:
       - Open System Settings
       - Privacy & Security → Microphone
       - Click the "+" button
       - Navigate to the .app bundle in DerivedData
       - Add it
       - Toggle ON
    
    d. As a last resort, use `tccutil reset Microphone` to clear 
       all mic permissions (Todd will have to re-grant for every 
       app, so use this only if other paths fail)

═══════════════════════════════════════════════════════════════════
TASK 2 — VERIFY MIC ACTUALLY WORKS
═══════════════════════════════════════════════════════════════════

Once permission is granted, run a single mic tap test:

2.1 Tap mic on the running Mac Catalyst app
2.2 Say a short phrase ("hello Baxter, what time is it")
2.3 Capture full Console output through the entire pipeline

Expected outcomes (any of these is a useful finding):

a. **Voice works on Mac.** Pipeline runs, response heard. This 
   means the iPad bug is iPad-specific (different from Mac 
   behavior). Capture the working Console log as a baseline.

b. **Same instant-fail on Mac as iPad.** Visible-grace fires on 
   tap, no listening. This is the most useful outcome — we now 
   have a fast iteration environment that reproduces the bug. 
   Capture the failing Console log as the diagnostic artifact.

c. **Different failure mode on Mac.** Something else goes wrong 
   (engine starts but nothing comes back, or some other error). 
   Capture and document what differs.

═══════════════════════════════════════════════════════════════════
TASK 3 — UPDATE deploy.sh OR DOCS FOR FUTURE MAC RUNS
═══════════════════════════════════════════════════════════════════

If launching from Xcode is the only reliable way, document this 
in the project README or a new MAC_DEBUGGING.md file. Future 
sessions need to know: "to debug voice on Mac, launch from Xcode 
with Mac Catalyst destination."

If a particular alternate path worked (Finder double-click, 
manual TCC add, etc.), document that.

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

App repo (~/Projects/home-control):
- No code changes expected. This is a configuration and 
  permission-grant task.
- IF code changes are needed (e.g. entitlements adjustment), 
  document them clearly. Keep them minimal.
- New MAC_DEBUGGING.md file documenting the working launch 
  procedure for future sessions.

Diagnostic artifact:
- Mac Console log from the first successful mic-tap (working 
  pipeline) OR the reproduced instant-fail (failing pipeline). 
  Posted to relay as kind=misc with title "mac-mic-tap-console-log".

REPORT MUST INCLUDE:
- Task 0 confirmation (relay URL + GitHub URL)
- Which launch path worked (Xcode debugger / Finder / open / 
  manual TCC / tccutil reset)
- Whether Todd had to grant permission via dialog or manual add
- Whether Mac voice now works OR reproduces the iPad bug
- Full Console log URL (posted as kind=misc to relay)
- iPad install confirmation OR confirmed-unreachable per standing 
  rule (no new build expected, but verify build 75 still on 
  iPads)
- Any architectural surprises

═══════════════════════════════════════════════════════════════════
NOTES TO TODD WHILE YOU WORK
═══════════════════════════════════════════════════════════════════

When you reach the point where Todd needs to interact (granting 
mic permission), pause and post a short message to the relay as 
kind=misc with title "mac-permission-needed" telling Todd what 
to do. Then continue once you see Todd has acted (you can poll 
to confirm by checking System Settings via shell or by re-trying 
mic).

If at any point you're stuck without Todd's interaction (TCC 
dialog won't appear, manual add not possible from CLI, etc.), 
post a kind=misc "mac-debugging-stuck" message describing 
exactly what state you're in and what Todd needs to do, then 
end the phase. Don't loop indefinitely waiting.

═══════════════════════════════════════════════════════════════════
WHAT NOT TO DO
═══════════════════════════════════════════════════════════════════

- Don't try to fix the instant-fail bug yet. The point of this 
  phase is to get a working Mac debug environment. Bug fix is 
  a separate phase that comes after we have logs.
- Don't add Mac-specific UI adjustments.
- Don't try to make Porcupine work on Mac.
- Don't try to make iCloud Shared Album work on Mac.
- Don't ship a new build to iPads. Build 75 stays on iPads. This 
  phase is Mac-only work.

POST WITH CORRECT HEADERS:
  curl -sS -X POST \
    -H "X-Auth-Token: $RELAY_TOKEN" \
    -H "X-Kind: completion" \
    -H "X-Title: phase-0c-24-mac-mic-permission" \
    -H "X-From-Instance: CC" \
    --data-binary @report.md \
    "$RELAY_URL/upload"

Commit to GitHub at:
completions/2026-04-29-phase-0c-24-mac-mic-permission.md
