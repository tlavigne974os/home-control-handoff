PHASE 0c.25 — iPad voice latency measurement (instrumented build)

Mac voice pipeline works end-to-end (0c-24, build 75). iPad voice 
"works" but is unusably slow: 7-9 seconds to first verbal tick, 
2-5 seconds more for response. Total: 9-14 seconds tap-to-response. 
That's not "talking to someone," that's voicemail.

This phase: ship an instrumented build to iPad, capture real timing 
data for a single turn, post the log back. We've done three rounds 
of code-reasoned latency fixes; this round is measurement-driven.

Mac is debugging environment, iPad is the actual product. Focus on 
iPad.

Task 0: Commit this prompt verbatim to BOTH the relay and the 
GitHub completion report URL before any other work.

═══════════════════════════════════════════════════════════════════
WHY MEASUREMENT NOW
═══════════════════════════════════════════════════════════════════

Mac numbers from 0c-24 verification:
- STT first partial: 2.43s
- LLM TTFT: 522ms
- (Total tap-to-response unmeasured but appeared reasonable)

iPad numbers from Todd's lived test (build 75):
- Tap to verbal tick: 7-9s
- Tick to response audio: 2-5s
- Total tap-to-response: 9-14s

The 4-7s gap between Mac and iPad first-audio is not yet explained. 
Reasoned-from-code suspects so far:
1. SFSpeechRecognizer initialization per-turn (1-1.5s) — recurs 
   every turn, may compound on iPad more than Mac
2. On-device STT model loading on iPad vs network STT on Mac
3. Tick firing AFTER STT finalize instead of in parallel — if this, 
   the tick is doing nothing for perceived latency
4. Something iPad-specific in audio session setup
5. SFSpeechAudioBufferRecognitionRequest configuration differences

We need real numbers, not more speculation.

═══════════════════════════════════════════════════════════════════
PART 1 — INSTRUMENT THE PIPELINE
═══════════════════════════════════════════════════════════════════

Add explicit timing prints with [Baxter][TIMING] prefix at every 
significant pipeline event. Each print should include:
- The stage name
- Absolute wall-clock time (from CFAbsoluteTimeGetCurrent or 
  Date().timeIntervalSince1970)
- Time-since-tap if a tap reference is established

Required instrumentation points (add wherever they don't already 
exist):

A. Mic tap fired — VoiceCoordinator.startConversation() entry
B. presentationState transitions — already has didSet from 0c.22, 
   confirm working
C. AppleSpeechCoordinator.startListening() entry
D. SFSpeechRecognizer instance creation
E. AVAudioEngine.start() returns
F. SFSpeechAudioBufferRecognitionRequest created
G. recognitionTask started (Apple Speech is now actively listening)
H. First interim result received (any non-empty transcript)
I. isFinal result received from STT
J. transcript handed off from AppleSpeechCoordinator to 
   VoiceCoordinator.runPipeline
K. Tick selected (voiceService.pickThinkingTick returns)
L. tickPlayer.play() called
M. tickPlayer audio actually starts (use AVAudioPlayer delegate or 
   best-effort: mark when play() returns)
N. assistant.streamResponse() invoked
O. CartesiaWSClient connect attempt (or "reusing persistent connection")
P. CartesiaWSClient connected (handshake complete or skipped if 
   reused)
Q. LLM first token received
R. First text chunk sent to Cartesia
S. First Cartesia audio chunk received
T. AudioStreamPlayer.start() called
U. First audio buffer scheduled to AVAudioEngine
V. First audio actually audible (best-effort: when first buffer 
   completion handler fires, OR mark when start() returns)
W. Stage 4 tick await begins (if applicable)
X. Stage 4 tick await ends
Y. Final LLM token received
Z. Last Cartesia audio chunk received
AA. AudioStreamPlayer.awaitCompletion() returns

Each event prints time-since-tap. For example:
  [Baxter][TIMING] +0.000s | mic tap fired
  [Baxter][TIMING] +0.087s | startListening entry
  [Baxter][TIMING] +0.342s | SFSpeechRecognizer created
  [Baxter][TIMING] +0.398s | AVAudioEngine.start() returned
  [Baxter][TIMING] +0.412s | recognitionTask started
  [Baxter][TIMING] +1.234s | first interim result
  [Baxter][TIMING] +3.567s | isFinal result received
  ...

If any step is conditional (e.g. "reusing persistent connection"), 
note both the path taken AND the time. So:
  [Baxter][TIMING] +3.612s | Cartesia: reusing persistent connection
or
  [Baxter][TIMING] +3.612s | Cartesia: opening new connection
  [Baxter][TIMING] +3.789s | Cartesia: handshake complete

═══════════════════════════════════════════════════════════════════
PART 2 — SHIP TO iPad
═══════════════════════════════════════════════════════════════════

This becomes build 76. Standing rule applies:

- Install on iPad Pro (8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2)
- Install on iPad Air (25423385-294F-50EA-8498-5690FECE21BF)
- If either is unreachable, document the unreachable state
- Don't skip silently

Per the standing rule from 0c.21 + 0c.22, deploy.sh's pre-flight 
check should catch unreachability. If it doesn't, fall back to 
manual install.

═══════════════════════════════════════════════════════════════════
PART 3 — CAPTURE A REAL TURN
═══════════════════════════════════════════════════════════════════

After build 76 is on iPad Pro and verified running:

3.1 Connect iPad Pro to Mac via USB cable

3.2 Open Console.app on Mac (Applications > Utilities > Console)

3.3 In Console.app, select the iPad Pro from the left sidebar 
    (under Devices). This shows iPad logs.

3.4 In the search box, filter on: [Baxter][TIMING]
    This filters to just the timing prints.

3.5 In the iPad app:
    - Wait for the app to be fully settled (no startup activity 
      logs)
    - Tap the mic button
    - Say a SHORT phrase: "what time is it"
    - Wait for the full response (Baxter says whatever, audio 
      stops)

3.6 In Console.app:
    - Wait until logs settle (no new lines for 5 seconds)
    - Select all the [Baxter][TIMING] lines for this turn
    - Copy them
    - Save to a file: ipad-timing-build76-turn1.log

3.7 Repeat steps 3.5-3.6 for two more turns:
    - Turn 2: same short phrase ("what time is it") to see if 
      timing changes after first turn (warm-up effects)
    - Turn 3: a slightly longer phrase ("Baxter, what's the 
      weather like outside")

3.8 Save all three logs.

═══════════════════════════════════════════════════════════════════
PART 4 — ANALYZE AND POST FINDINGS
═══════════════════════════════════════════════════════════════════

Look at the three logs. For each, identify:

1. Where the largest gaps between events are (the slow stages)
2. Whether the timing is consistent across turns or changes (e.g. 
   first turn slower than turn 2/3 — indicates initialization cost)
3. Whether the tick is firing parallel with LLM/STT or sequentially
4. Where the 7-9s "first audio" delay actually accumulates from

═══════════════════════════════════════════════════════════════════
DELIVERABLE — CRITICAL: TWO ARTIFACTS TO POST
═══════════════════════════════════════════════════════════════════

Post TWO things to the relay:

A. The completion report (analysis + recommended fixes):
   curl -sS -X POST \
     -H "X-Auth-Token: $RELAY_TOKEN" \
     -H "X-Kind: completion" \
     -H "X-Title: phase-0c-25-ipad-latency-measurement" \
     -H "X-From-Instance: CC" \
     --data-binary @completion-report.md \
     "$RELAY_URL/upload"

B. The raw timing logs (three turns concatenated):
   curl -sS -X POST \
     -H "X-Auth-Token: $RELAY_TOKEN" \
     -H "X-Kind: misc" \
     -H "X-Title: ipad-timing-logs-build76" \
     -H "X-From-Instance: CC" \
     --data-binary @ipad-timing-all-turns.log \
     "$RELAY_URL/upload"

The completion report should reference the raw log URL.

REPORT MUST INCLUDE:
- Task 0 confirmation (relay URL + GitHub URL)
- Confirmation build 76 deployed on both iPads (or unreachable 
  state)
- Three turns captured with timestamps
- Analysis: where is the 7-9s actually accumulating?
- Specifically: is the tick firing parallel with LLM, or sequential?
- Specifically: is per-turn STT initialization the culprit (first 
  turn vs turn 2/3 timing comparison)?
- Specifically: did the persistent CartesiaWS reuse work, or is 
  it reconnecting every turn?
- Recommended fixes ranked by likely impact (don't implement yet)
- Whether Apple STT itself is the bottleneck or something else
- Raw log URL on relay

═══════════════════════════════════════════════════════════════════
DO NOT IMPLEMENT FIXES IN THIS PHASE
═══════════════════════════════════════════════════════════════════

This phase is measurement only. Resist the urge to ship a fix in 
the same build as the instrumentation. We've done three rounds of 
"diagnose AND fix in one phase" and they keep producing partial 
results because the fix is informed by reasoning, not measurement.

Once the data is in, DC and Todd will choose the next phase based 
on what the numbers actually show. That phase ships the fix.

═══════════════════════════════════════════════════════════════════
IF iPad PRO IS UNREACHABLE
═══════════════════════════════════════════════════════════════════

If the iPad Pro is offline / asleep / not on WiFi when you try to 
install:

1. Per standing rule, document the unreachable state
2. Post a kind=misc message to the relay titled 
   "ipad-pro-unreachable-need-todd" with what state it's in
3. End the phase there. Don't loop trying. Don't fall back to 
   simulator (the instrumentation needs real iPad data).

Todd will wake/connect the iPad and the phase resumes when he 
posts back.

═══════════════════════════════════════════════════════════════════
ARCHITECTURAL NOTE
═══════════════════════════════════════════════════════════════════

If the timing data shows that the bulk of the latency is in Apple 
SFSpeechRecognizer (STT itself), that's a real architectural 
finding. Don't try to fix it in this phase. We may need to 
evaluate alternatives (Deepgram, AssemblyAI, OpenAI Whisper API) 
in a future phase. Capture the finding clearly so the decision 
can be made with data.

If the timing data shows the bottleneck is somewhere else 
(Cartesia connection, audio session setup, tick gating), the fix 
is likely much smaller and stays inside the existing architecture.

The data will tell us. Don't pre-judge.

═══════════════════════════════════════════════════════════════════
GITHUB COMPLETION
═══════════════════════════════════════════════════════════════════

Commit completion report to GitHub at:
completions/2026-04-29-phase-0c-25-ipad-latency-measurement.md
