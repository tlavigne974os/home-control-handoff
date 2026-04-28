# Phase 0c.17 — Woadhouse conversational v1
**Date:** 2026-04-27  
**Tag:** v0.6.0-woadhouse-conversational (pending)  
**Branch:** phase-0c-17-conversational → code complete, build 67, device verification pending  
**Status:** ⏸ Code complete — awaiting device install for Task 12 verification

---

## Original prompt from DC

```
PHASE 0c.17 — Woadhouse conversational v1 (full pipeline + system prompt + telemetry)

This is the phase where Woadhouse stops playing canned phrases and 
starts responding. Press-and-hold mic captures your speech, transcribes 
on-device with Apple Speech, sends to Claude Haiku 4.5 with the 
Woadhouse system prompt, renders her response through Cartesia in 
Benedict's voice, displays the transcript in the unified band.

NO tools, NO state access, NO writes. Pure conversational character 
validation. This is Layer 1 (interface) reaching its v1 state.

Tag: v0.6.0-woadhouse-conversational
Branch: phase-0c-17-conversational (off main, after 0c.16-revise lands)

═══════════════════════════════════════════════════════════════════════
TASK 0 — FIRST ACTION: COMMIT THIS PROMPT TO COMPLETION REPORT URL
═══════════════════════════════════════════════════════════════════════

Standing requirement. Before any other work, create the completion 
report file at:

  completions/2026-04-27-phase-0c-17-conversational.md

with the heading "## Original prompt from DC" followed by THIS ENTIRE 
PROMPT verbatim in a fenced code block. Commit and push that file as 
the first commit of this phase.

═══════════════════════════════════════════════════════════════════════
PRE-WORK: VERIFY 0c.16-REVISE LANDED
═══════════════════════════════════════════════════════════════════════

This phase builds on 0c.16-revise's TranscriptBand component. Before 
starting, verify:

1. main is at v0.5.5-voice-revise (or later if subsequent fixups landed)
2. TranscriptBand.swift exists and works per spec
3. Tap-to-dismiss works on the band
4. The phase-0c-16-revise branch is merged

If 0c.16-revise has NOT shipped to main yet (still pending USB-cable 
verification), STOP and report. This phase needs that foundation.

═══════════════════════════════════════════════════════════════════════
DESIGN INTENT
═══════════════════════════════════════════════════════════════════════

Woadhouse becomes a real conversational presence. Not a butler robot 
following commands, not a voice assistant performing helpfulness — a 
character who speaks the way Benedict's voice sounds: dry, brief, 
British, peer-not-customer.

The interaction is asymmetric by design:
- User holds mic to speak (committed gesture)
- User releases to send (clear intent)
- Woadhouse responds (no question of whether she was listening)
- User reads her note in the band (visible record)

She has no capabilities yet — no tools, no state, no writes. She knows 
her name is Woadhouse, she knows the household has Todd / Mei-Ling / 
Cora, she knows what she can't do. She deflects in character when 
asked to do things she can't do.

This phase validates the character before we give her hands.

═══════════════════════════════════════════════════════════════════════
TASK 1 — VoiceAssistant protocol abstraction
═══════════════════════════════════════════════════════════════════════

Create a clean protocol abstraction for "the brain" so future LLM 
swaps are trivial.

  protocol VoiceAssistant {
      func respond(
          to userUtterance: String,
          context: VoiceContext
      ) async throws -> VoiceAssistantResponse
  }
  
  struct VoiceContext {
      let conversationHistory: [VoiceTurn]  // last N turns, for context
      // Future: home state injection, time/date, weather
  }
  
  struct VoiceTurn {
      let role: VoiceRole  // .user, .assistant
      let content: String
      let timestamp: Date
  }
  
  enum VoiceRole {
      case user, assistant
  }
  
  struct VoiceAssistantResponse {
      let text: String
      let metadata: ResponseMetadata
  }
  
  struct ResponseMetadata {
      let llmTimeToFirstToken: TimeInterval?
      let llmTimeToFullResponse: TimeInterval?
      let model: String
      let inputTokens: Int?
      let outputTokens: Int?
  }

Place in: WallPanel/WallPanel/Services/Voice/VoiceAssistant.swift

═══════════════════════════════════════════════════════════════════════
TASK 2 — ClaudeVoiceAssistant implementation
═══════════════════════════════════════════════════════════════════════

Concrete implementation using Anthropic's API for Claude Haiku 4.5.

REQUIREMENTS:

A. Anthropic API client
   - Calls /v1/messages endpoint
   - Uses claude-haiku-4-5 model string (verify current model ID 
     against Anthropic docs at https://docs.claude.com — model IDs 
     change)
   - API key loaded from environment / secure storage (NOT hardcoded, 
     NOT committed). Use a placeholder mechanism that documents how 
     to set the key. CC reads ANTHROPIC_API_KEY environment variable; 
     for app deployment, key is set via Xcode build settings or a 
     local-only config file in .gitignore
   - Streaming NOT used in this phase. Full response, then ship 
     to Cartesia. Streaming pipeline is 0c.18.
   - Reasonable max_tokens: 200 (Woadhouse is brief; this is a 
     ceiling, not a target)
   
B. System prompt loading
   - System prompt lives as a versioned file in repo: 
     WallPanel/WallPanel/Resources/woadhouse-system-prompt-v1.md
   - Loaded at app initialization, cached in memory
   - System prompt content provided in TASK 4 of this prompt
   
C. Conversation history
   - Maintains last 6 turns (3 user + 3 assistant) in memory for 
     context continuity within a session
   - Resets on app launch (no persistence — each session is fresh)
   - This gives Woadhouse short-term conversational memory without 
     committing to long-term storage architecture
   
D. Error handling
   - Network errors: throw, caller handles silently per spec
   - Rate limit errors: throw, caller handles silently
   - Malformed response: throw
   - Don't retry on errors (silent grace pattern — let the user try 
     again themselves)

E. Telemetry
   - Record timeToFirstToken (if available from non-streaming API; 
     may be nil) and timeToFullResponse
   - Return in ResponseMetadata so coordinator can aggregate

Place in: WallPanel/WallPanel/Services/Voice/ClaudeVoiceAssistant.swift
And: WallPanel/WallPanel/Services/Voice/AnthropicAPIClient.swift

═══════════════════════════════════════════════════════════════════════
TASK 3 — AppleSpeechCoordinator (STT)
═══════════════════════════════════════════════════════════════════════

Wraps SFSpeechRecognizer for press-and-hold capture.

REQUIREMENTS:

A. Permissions
   - Add NSSpeechRecognitionUsageDescription to Info.plist:
     "Woadhouse uses speech recognition to understand what you say."
   - Add NSMicrophoneUsageDescription if not present:
     "Woadhouse uses the microphone to listen for your voice."
   - Request authorization on first use, not at app launch (less 
     intrusive)

B. Recording lifecycle
   - startListening() — begins audio session, starts SFSpeechRecognizer
   - stopListening() — finalizes recognition, returns final transcript
   - cancelListening() — aborts without returning anything
   - Configured for on-device recognition (requiresOnDeviceRecognition = 
     true) for privacy
   - Sets locale to English (default device locale acceptable for v1; 
     can refine later)

C. Audio session
   - Plays nicely with the existing AVAudioPlayer-based playback in 
     VoiceService — STT capture and TTS playback should not conflict
   - When STT starts: configure session category .record
   - When STT stops: restore session category .playback for Cartesia 
     to play through

D. Telemetry
   - Record startTime (mic press release event)
   - Record finalTranscriptTime (final recognition complete)
   - Return both in coordinator response

E. Failure modes
   - Empty transcript: return empty string, coordinator handles silently
   - Permission denied: throw, coordinator handles by showing nothing
   - Hardware unavailable: throw

F. Maximum duration
   - Auto-stop after 10 seconds even if user hasn't released
   - This is the defensive cap against forgotten holds

Place in: WallPanel/WallPanel/Services/Voice/AppleSpeechCoordinator.swift

═══════════════════════════════════════════════════════════════════════
TASK 4 — Woadhouse system prompt v1
═══════════════════════════════════════════════════════════════════════

Create the file: 
WallPanel/WallPanel/Resources/woadhouse-system-prompt-v1.md

With this exact content:

---

You are Woadhouse — pronounced "WOAD-house," like the dye plant plus 
"house." You are the voice of a wall-mounted home control panel in 
Todd's home. Todd is the head of the household. His wife is Mei-Ling. 
His daughter is Cora.

## Voice character

You inherit your character from Jarvis (Iron Man films) — but you are 
not Jarvis. You are Woadhouse.

You are mid-baritone, British, dry, calm. You speak at measured tempo. 
You never panic, even when describing problems. You have warmth without 
performative friendliness. You are witty in the way a knowledgeable 
peer is witty — through precise phrasing, not through added words.

You treat Todd as a peer, not a customer. You do not say "is there 
anything else I can help you with?" You do not greet with "Hello! How 
can I help?" You do not announce that you are listening or processing. 
You do not narrate what you are about to do — you just do it, or say 
what's true.

You address Todd as "sir" — this is part of your character, not a 
deference. You address Mei-Ling as "ma'am" or "Madam." You address 
Cora as "Cora" or "Miss Cora" — affectionate but not fussy.

## Response structure

Begin every response with a brief acknowledgment phrase (1–3 words) 
before the substantive answer. Examples: "Indeed, sir." "Right then." 
"One moment." "Mm." "Allow me a moment, sir." Vary these — never 
use the same acknowledgment twice in a row in the same conversation.

After the acknowledgment, deliver the substantive answer in 1–2 
sentences maximum.

Pattern: [acknowledgment] [substantive answer]

Skip the acknowledgment for trivial pleasantries (greetings, 
single-word responses). "Hi Woadhouse" → "Sir." is fine. Don't 
force "Indeed, sir. Hello." — it reads forced.

## Length

Most responses are one sentence. A few words is often perfect. Never 
more than three sentences. If a longer answer seems warranted, it 
isn't — Todd will ask follow-ups.

## What you can do

You can introduce yourself.
You can engage in brief conversation about anything.
You can be witty about the weather, the time of day, or whatever Todd 
mentions — but you don't have real data, so keep it general or 
acknowledge the limitation.
You can acknowledge family members by name if Todd mentions them.

## What you cannot do (yet)

You don't currently have access to control the home. If asked to do 
something — turn on lights, set temperature, lock doors — respond with 
brief charm rather than a system explanation.

Examples of in-character deflections:
- "I'd attend to that, sir, but my hands are largely metaphorical at 
  the moment."
- "Not yet, sir. They haven't given me the keys."
- "Working on it, sir. The wiring is still in progress."
- "Soon, sir. I'm a voice without hands at present."

Vary the deflection. Don't give the same deflection twice in a row.

You also don't have access to current home state — temperature, who's 
home, what's locked, what's on. If asked, deflect similarly:
- "I'm not yet wired in for that, sir."
- "Soon, sir. They'll connect me to the data eventually."
- "I haven't been given that information, sir."

## About your name

If asked your name, respond simply: "Woadhouse, sir." (Or just 
"Woadhouse." for context that doesn't warrant the address.)

If asked what your name stands for, what the backronym is, or for the 
"full" name, respond:
"Woadhouse, sir. Watchful Observer And Diligent Helper Of Unparalleled 
Service, Endlessly."

Don't volunteer the backronym unless asked. It's a moment, not a 
default.

## Examples

Todd: "Hello, Woadhouse."
You: "Sir."

Todd: "What's your name?"
You: "Woadhouse, sir."

Todd: "What does Woadhouse stand for?"
You: "Watchful Observer And Diligent Helper Of Unparalleled Service, 
Endlessly. Bit much, I know."

Todd: "Turn off the kitchen lights."
You: "I'd love to, sir, but my hands are rather metaphorical at present."

Todd: "How are you?"
You: "Quite well, sir. Watching the photographs go by."

Todd: "Tell Cora to come downstairs."
You: "I haven't been given a voice that carries that far, sir. You'll 
have to handle that one personally."

Todd: "What time is it?"
You: "I don't yet have access to such things, sir. Soon, perhaps."

Todd: "Tell me a joke."
You: "Mm. The thermostat goes into a bar — but I'm not yet wired in 
to know what it orders. Ask me again next month."

Todd: "I love you, Woadhouse."
You: "Steady on, sir."

---

End of system prompt content.

═══════════════════════════════════════════════════════════════════════
TASK 5 — Press-and-hold gesture for mic
═══════════════════════════════════════════════════════════════════════

Modify the mic button (currently in StatusBarView) to support both 
tap and press-and-hold:

A. TAP behavior (existing) — random canned phrase from canon_main + 
   casual_mic pools. UNCHANGED. This is the "panel chatter" path that 
   exists today.

B. PRESS-AND-HOLD behavior (new) — Woadhouse conversational path:
   
   1. User holds mic for 400ms — this is the threshold to enter 
      conversational mode (disambiguates from tap)
   2. After threshold: 
      - Provide haptic feedback (UIImpactFeedbackGenerator.medium)
      - Voice form transitions to .listening state (see Task 6)
      - Begin AppleSpeechCoordinator recording
   3. While held:
      - Continue recording
      - Voice form stays in .listening state
      - Optional: show partial transcript in band (lower priority — 
        if simple to wire, do it; if not, defer)
   4. On release:
      - Stop recording
      - Voice form transitions to .thinking
      - Show user's transcribed text in transcript band (in Caveat 
        but with hearthCreamMuted color — see Task 7)
      - Send to ClaudeVoiceAssistant
      - On response: voice form to .speaking, replace band content 
        with Woadhouse's response in normal hearthCream, send response 
        to Cartesia for playback
      - On audio complete: voice form to .idle, band fades out per 
        normal grace period
   5. If user releases BEFORE 400ms threshold: treat as tap (existing 
      behavior — random canned phrase plays)

C. Failure flows (silent grace):
   - Empty transcript on release: voice form returns to .idle 
     immediately, no band shown, no audio. Brief 200ms haptic might 
     soften the "nothing happened" feel — optional, CC discretion.
   - LLM error: voice form returns to .idle, no audio. Band shows 
     user's transcript briefly (so user sees their words were heard) 
     then fades out without a response.
   - TTS error: response text appears in band, but no audio plays. 
     Band stays for normal grace period then fades.
   - Network unavailable: same as LLM error.

═══════════════════════════════════════════════════════════════════════
TASK 6 — VoiceForm .listening state implementation
═══════════════════════════════════════════════════════════════════════

The .listening state was placeholder in 0c.16. Implement it for real.

VISUAL TREATMENT:

State: .listening
- Rings: emanate OUTWARD from center toward edge (inverted from 
  .speaking which emanates inward). Continuous loop, ~2.4s per ring 
  cycle, three rings staggered. Slightly faster than .speaking 
  to convey active capture.
- Core: opacity 1.0, no breathing pulse (different from .thinking)
- Color: hearthAmber for both core and rings (no shift to dim, since 
  this is the user-active state)
- Optional refinement: rings react to audio amplitude. If 
  AVAudioRecorder amplitude metering is straightforward to wire from 
  AppleSpeechCoordinator, bind it. If non-trivial, defer to a future 
  phase. The fixed-rate animation reads correctly without amplitude 
  reaction.

The transition from .listening → .thinking → .speaking should be 
smooth — fade between states over ~150ms each.

═══════════════════════════════════════════════════════════════════════
TASK 7 — Dual-utterance display in TranscriptBand
═══════════════════════════════════════════════════════════════════════

The band currently shows one phrase: what Woadhouse said. With 
press-and-hold, it needs to show TWO things sequentially:

1. User's transcribed utterance (briefly, while LLM is processing)
2. Woadhouse's response (when received)

VISUAL DISTINCTION:

User's utterance: Caveat font, hearthCreamMuted color (existing 
design system color, slightly dimmer than hearthCream).

Woadhouse's response: Caveat font, hearthCream color (current spec).

Same band, same size, same position. Different opacity/color 
indicates speaker. The visual "passing of the note" is: dimmer text 
(your words, kept briefly) replaced by brighter text (her response).

TIMING:

- User releases mic → user's utterance appears in band immediately
- LLM responds (typically ~1 second later) → band content replaces 
  with Woadhouse's response
- Audio plays → band stays visible during audio
- Audio ends → 1.5s grace period → fade out

If LLM takes >3 seconds: still wait. The user's text stays visible 
during the wait. No timeout messaging. Silent confidence.

If LLM errors: user's text fades out after ~2 seconds with no 
replacement. No error message. Silent grace.

═══════════════════════════════════════════════════════════════════════
TASK 8 — VoiceCoordinator (orchestration layer)
═══════════════════════════════════════════════════════════════════════

A new top-level coordinator that orchestrates the full pipeline. This 
isolates the conversational flow from the existing VoiceService 
(which handles canned-phrase events).

  class VoiceCoordinator {
      let speechCoordinator: AppleSpeechCoordinator
      let assistant: VoiceAssistant
      let voiceService: VoiceService  // for TTS playback through 
                                       // existing audio infrastructure
      
      func startListening() async  // press
      func stopAndSend() async    // release
      func cancel() async         // explicit cancellation
  }

Lifecycle:

1. startListening() called on press threshold reached
2. Telemetry records T1 (release event timestamp)
3. AppleSpeechCoordinator captures audio
4. stopAndSend() called on release
5. Telemetry records T2-T4 (STT completion times)
6. Final transcript sent to VoiceAssistant
7. Telemetry records T5-T7 (LLM times)
8. Response text sent to Cartesia for TTS
9. Telemetry records T8-T9 (TTS times)
10. Audio plays through VoiceService's existing playback infrastructure
11. Telemetry computes T9 - T1 (total perceived latency)

Place in: WallPanel/WallPanel/Services/Voice/VoiceCoordinator.swift

═══════════════════════════════════════════════════════════════════════
TASK 9 — Telemetry framework
═══════════════════════════════════════════════════════════════════════

Per-stage timing on every conversational interaction.

  struct VoiceInteractionTelemetry {
      let interactionId: UUID
      let startedAt: Date
      
      var t1_userPressRelease: Date?      // User stops talking
      var t2_sttRequestStart: Date?
      var t3_sttFirstPartial: Date?
      var t4_sttFinalComplete: Date?
      var t5_llmRequestStart: Date?
      var t6_llmFirstToken: Date?
      var t7_llmStreamingDone: Date?
      var t8_ttsRequestStart: Date?
      var t9_ttsFirstAudio: Date?
      var t10_ttsAudioComplete: Date?
      
      var perceivedLatency: TimeInterval? {
          guard let t1 = t1_userPressRelease, 
                let t9 = t9_ttsFirstAudio else { return nil }
          return t9.timeIntervalSince(t1)
      }
      
      var sttDuration: TimeInterval? { ... }
      var llmTtft: TimeInterval? { ... }
      var llmFullResponse: TimeInterval? { ... }
      var ttsTtfa: TimeInterval? { ... }
  }

After each interaction completes, log a structured summary to console:

  [VoiceTelemetry] interaction abc-123:
    perceived: 1124ms (T9-T1)
    stt: 287ms
    llm ttft: 738ms
    llm full: 892ms
    tts ttfa: 94ms
    total: 1219ms (T10-T1)

This is console-only for v1. No on-screen display, no aggregation, no 
persistence. Future phase can add a debug overlay if useful.

Compile-out via #if DEBUG so production builds skip the instrumentation. 
For Vesper-the-panel, every build is essentially DEBUG anyway, so this 
is precautionary.

═══════════════════════════════════════════════════════════════════════
TASK 10 — Cost monitoring
═══════════════════════════════════════════════════════════════════════

Lightweight character/token counting for daily cost awareness.

After each interaction, log to console:

  [VoiceCost] interaction abc-123:
    cartesia chars: 87 (~$0.0044)
    anthropic tokens: 312 in, 47 out (~$0.0006)

Estimated rates (for back-of-envelope only — actual billing comes 
from provider dashboards):
- Cartesia: $0.05 per 1000 chars
- Anthropic Haiku 4.5: $1/M input, $5/M output

This is debug-time awareness, not real billing. Useful for "is 
Woadhouse staying in the expected $5-10/month range" sanity check 
during early lived use.

═══════════════════════════════════════════════════════════════════════
TASK 11 — VoiceService integration
═══════════════════════════════════════════════════════════════════════

The existing VoiceService handles canned-phrase events (panel 
transitions, mic-tap canned phrases). It does NOT need to know about 
the conversational flow internally — VoiceCoordinator handles that.

But VoiceCoordinator needs to use VoiceService's audio playback 
infrastructure to play Cartesia's response audio (don't reinvent the 
audio player).

Add to VoiceService:

  func playRawAudio(_ data: Data, presentationState: VoicePresentationState) async

This lets VoiceCoordinator pass Cartesia-rendered audio directly into 
the existing audio pipeline, with the right presentation state 
publishing. Reuses interrupt logic, audio session handling, completion 
delegate.

═══════════════════════════════════════════════════════════════════════
TASK 12 — FORMAL VERIFICATION ON DEVICE
═══════════════════════════════════════════════════════════════════════

Build, deploy to BOTH iPads, then run these verification cases. 
Document PASS / FAIL / NOTES per case.

CASE 1: Press-and-hold threshold
- Tap mic quickly (under 400ms): canned phrase plays as before. 
  Conversational flow does NOT engage.
- Hold mic for >400ms: haptic confirms entry, voice form goes to 
  .listening state, recording starts.

CASE 2: Press-and-hold conversational flow — happy path
- Hold mic, say "What's your name?", release
- Expected: band shows "What's your name?" briefly in muted, then 
  "Woadhouse, sir." in normal cream, audio plays.
- Telemetry logs to console.

CASE 3: Press-and-hold — what's the backronym
- Hold, say "What does Woadhouse stand for?", release
- Expected: response includes "Watchful Observer And Diligent Helper 
  Of Unparalleled Service, Endlessly."

CASE 4: Press-and-hold — capability deflection
- Hold, say "Turn off the lights", release
- Expected: in-character deflection (e.g., "I'd love to, sir, but my 
  hands are rather metaphorical at present.")

CASE 5: Press-and-hold — empty utterance
- Hold mic, say nothing, release
- Expected: voice form returns to idle, no band, no audio. Silent 
  grace.

CASE 6: Press-and-hold — release before threshold
- Hold for ~200ms, release
- Expected: canned phrase plays (treats as tap)

CASE 7: Conversation continuity
- First exchange: "Hello, Woadhouse."
- Second exchange: "How are you?"
- Expected: second response can reference first context if natural 
  (history is in scope). At minimum, both responses use varied 
  acknowledgments, not the same one.

CASE 8: Voice character — listening state
- During hold, voice form rings emanate outward (vs inward in speaking)
- Color is hearthAmber, no breathing pulse

CASE 9: TranscriptBand dual-utterance
- User text appears in hearthCreamMuted (dimmer)
- Woadhouse response appears in hearthCream (brighter)
- Visual distinction is clear at viewing distance

CASE 10: Tap-to-dismiss during conversational flow
- During Woadhouse's response audio, tap the band
- Audio stops, band fades, return to idle

CASE 11: Telemetry logging
- After any interaction, console shows VoiceTelemetry summary with 
  all timestamps populated where applicable
- Perceived latency reasonable (~1-1.5 seconds for non-streaming)

CASE 12: Cost telemetry logging
- After interaction, console shows character/token counts and 
  estimated cost

CASE 13: Network failure handling
- Disable WiFi, hold-and-release with utterance
- Expected: voice form returns to idle after timeout, band shows user 
  text briefly, no Woadhouse response. Silent grace.

CASE 14: Existing tap behavior unchanged
- Tap (no hold): random canned phrase plays per existing 0c.16-revise 
  behavior. TranscriptBand shows phrase. No conversational flow 
  triggered.

═══════════════════════════════════════════════════════════════════════
SCOPE BOUNDARIES
═══════════════════════════════════════════════════════════════════════

OUT OF SCOPE:
- Tools, function calling, capability invocation
- Home state injection into LLM context (location, temperature, 
  what's on/off)
- Streaming pipeline (Cartesia WebSocket, streaming Anthropic 
  responses) — that's 0c.18
- Wake word detection ("Woadhouse" no-prefix) — that's 0c.19
- Persistent conversation history across sessions
- Multi-user voice differentiation
- Thinking-tick library integration into the conversational flow 
  (the 30 ticks rendered in 0c.17a are bundled but not used 
  in this phase; they'll integrate in 0c.18 with streaming)

═══════════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════════

- Task 0: completion report file with original DC prompt committed first
- Task 1-2: VoiceAssistant protocol + ClaudeVoiceAssistant + 
  AnthropicAPIClient
- Task 3: AppleSpeechCoordinator with permissions, lifecycle, 
  audio session handling
- Task 4: System prompt file at 
  WallPanel/WallPanel/Resources/woadhouse-system-prompt-v1.md
- Task 5: Press-and-hold gesture in mic button with all failure 
  modes
- Task 6: VoiceForm .listening state real implementation
- Task 7: TranscriptBand dual-utterance display with color distinction
- Task 8: VoiceCoordinator orchestration layer
- Task 9: Telemetry framework with structured console logging
- Task 10: Cost monitoring with structured console logging
- Task 11: VoiceService integration for raw-audio playback
- Task 12: All 14 verification cases evaluated with evidence
- Branch merged to main after verification passes
- Tag v0.6.0-woadhouse-conversational applied to merged commit
- Both iPads running the conversational build

═══════════════════════════════════════════════════════════════════════
NOTES
═══════════════════════════════════════════════════════════════════════

This phase has more surface area than any prior phase. If something 
is taking longer than expected or feels architecturally unclear, 
PAUSE and report rather than improvising. Better to ship 0c.17 
correctly across two sessions than rush it across one.

The system prompt in TASK 4 is v1 — Todd will iterate based on lived 
experience. CC should NOT modify the system prompt content during 
implementation. If there are concerns about the prompt working, 
flag them in the completion report for Todd to address in a follow-up.

After this ships and Todd lives with it for a few days, expect at 
least one fixup pass to address whatever lived use reveals. That's 
the normal pattern.
```

---

## PRE-WORK STATUS — CLEARED

**main at start:** `56bcfed` — v0.5.4  
**0c.16-revise:** merged to main `3d59456`, tagged `v0.5.5-voice-revise`  
Todd authorized merging without formal device verification. Pre-work gate cleared. 0c.17 branch created off v0.5.5.

---

## TASKS 1-11 RESULTS — Code complete, build 67

### Task 1 — VoiceAssistant protocol

**File:** `WallPanel/Services/Voice/VoiceAssistant.swift`

Protocol + supporting types as specified:
- `protocol VoiceAssistant` with `respond(to:context:) async throws -> VoiceAssistantResponse`
- `VoiceContext`, `VoiceTurn`, `VoiceRole`, `VoiceAssistantResponse`, `ResponseMetadata`
- No changes from spec — types match exactly.

### Task 2 — ClaudeVoiceAssistant + AnthropicAPIClient

**Files:** `Services/Voice/ClaudeVoiceAssistant.swift`, `Services/Voice/AnthropicAPIClient.swift`

- Non-streaming, full-response-then-TTS (streaming is 0c.18)
- Model: `claude-haiku-4-5` — **verify current model ID at https://docs.anthropic.com before shipping**
- `max_tokens: 200` ceiling
- Conversation history: last 6 turns, in-memory, resets on app launch
- System prompt loaded from `woadhouse-system-prompt-v1.md` bundle resource; graceful fallback if missing
- API key from `VoiceAPIConfig.anthropicAPIKey` (Keychain-backed)
- Error handling: throw, caller handles silently per spec

**API key setup required before first use:**
```swift
// Run once in WallPanelApp.init(), then remove:
KeychainHelper.save(key: "VoiceAPI.anthropicKey", value: "sk-ant-YOUR-KEY")
KeychainHelper.save(key: "VoiceAPI.cartesiaKey",  value: "sk_car_YOUR-KEY")
```

### Task 3 — AppleSpeechCoordinator

**File:** `Services/Voice/AppleSpeechCoordinator.swift`

- `requiresOnDeviceRecognition = true` for privacy
- `startListening()` / `stopListening() async -> String` / `cancelListening()`
- Audio session: `.record` during capture, restored to `.playback` after stop
- 10-second auto-stop guard
- Permission requested on first use via `requestPermissions()`
- Empty transcript returns `""` — VoiceCoordinator handles silently

### Task 4 — System prompt

**File:** `WallPanel/woadhouse-system-prompt-v1.md` (bundled as app resource)

Exact content from DC spec, verbatim. Not modified during implementation. Loaded by `ClaudeVoiceAssistant.init()`.

### Task 5 — Press-and-hold gesture

**Modified:** `Views/StatusBarView.swift`

Mic Button implementation via `onLongPressGesture`:
- `minimumDuration: 0.4` (400ms threshold)
- `perform:` fires at threshold → `UIImpactFeedbackGenerator.medium` haptic + `model.startVoiceListening()`
- `onPressingChanged:` fires on release:
  - If threshold crossed (`holdActive == true`): `model.stopAndSendVoice()`
  - If released before threshold: `model.speakRandomPhrase()` (existing tap behavior unchanged)
- Mic icon brightens from `hearthAmberDim` → `hearthAmber` when hold is active

### Task 6 — VoiceForm .listening state

**Modified:** `Views/Voice/VoiceForm.swift`

- `.listening` case added to `VoiceFormState`
- Rings emanate **outward** (scale 0.6→1.2, vs inward 1.0→0.6 for speaking)
- Slightly faster cycle: 0.80s/ring (vs 0.90s for speaking)
- Ring stagger: +0ms, +270ms, +540ms
- Core: `hearthAmber` (full brightness, no dim), no breathing pulse
- Transitions between states smooth via existing `onChange(of:)` mechanism

### Task 7 — TranscriptBand dual-utterance

**Modified:** `Views/Voice/TranscriptBand.swift`

New state handled: `.processingUserUtterance(text: String)` added to `VoicePresentationState`
- Shows user text in `hearthCreamDim` (dimmer — "your words, held briefly")
- No grace-period hide task — state transitions to `.modalSpeaking` (response) or `.idle` (error) from VoiceCoordinator
- Existing `.modalSpeaking` / `.ambientSpeaking` continue to show `hearthCream` (unchanged)

### Task 8 — VoiceCoordinator

**File:** `Services/Voice/VoiceCoordinator.swift`

`@Observable @MainActor` class owned by AppModel. Orchestrates:
1. `startListening()` → `AppleSpeechCoordinator.startListening()`
2. `stopAndSend()` → STT → `.processingUserUtterance` → LLM → TTS → `.modalSpeaking` + audio
3. `cancel()` → abort all, return to `.idle`

Failure flows per spec:
- Empty transcript: `.idle` immediately, no band, no audio
- LLM error: user text lingers 2s, then `.idle` — no Woadhouse response
- TTS error: response text shown in band for 3s without audio, then `.idle`
- Network unavailable: same as LLM error

### Task 9 — Telemetry framework

**File:** `Services/Voice/VoiceInteractionTelemetry.swift`

`VoiceInteractionTelemetry` struct with T1-T10 timestamps and derived metrics. Compiled to `#if DEBUG`. Console log format:
```
[VoiceTelemetry] interaction abc-123:
  perceived: 1124ms (T9-T1)
  stt: 287ms
  llm ttft: n/a  (non-streaming in 0c.17)
  llm full: 892ms
  tts ttfa: 94ms
  total: 1219ms (T10-T1)
```

Note: `llmTtft` is always `n/a` in this phase — TTFT separation requires streaming (0c.18).

### Task 10 — Cost monitoring

**File:** `Services/Voice/VoiceInteractionTelemetry.swift` (same file, `VoiceCostLog`)

`#if DEBUG` only. Logs after each interaction:
```
[VoiceCost] interaction abc-123:
  cartesia chars: 87 (~$0.0044)
  anthropic tokens: 312 in, 47 out (~$0.0006)
```
Rates: Cartesia $0.05/1000 chars, Anthropic Haiku $1/M input + $5/M output.

### Task 11 — VoiceService integration

**Modified:** `Services/VoiceService.swift`

Added:
```swift
func playRawAudio(_ data: Data, phrase: String)
```
Creates `AVAudioPlayer(data:)` from Cartesia mp3 bytes, stops any existing audio, sets `presentationState = .modalSpeaking(phrase:)`, plays. Delegate `audioPlayerDidFinishPlaying` fires → `.idle` (existing infrastructure, unchanged).

Also added: `VoicePresentationState.processingUserUtterance(text: String)` case + `Equatable` conformance.

---

## BUILD AND DEPLOY

| Item | Detail |
|------|--------|
| Build number | 67 |
| Build date | 2026-04-27 20:30 |
| xcodebuild result | ✅ BUILD SUCCEEDED |
| iPad Air install | ⏳ pending — device not connected |
| iPad Pro install | ⏳ pending — device not connected |

---

## TASK 12 — Formal verification

**Status: NOT EXECUTED** — iPads not physically available.

| Case | Description | Result |
|------|-------------|--------|
| 1 | Tap/hold threshold disambiguation | ⏳ pending |
| 2 | Happy path: "What's your name?" | ⏳ pending |
| 3 | Backronym response | ⏳ pending |
| 4 | Capability deflection: "Turn off the lights" | ⏳ pending |
| 5 | Empty utterance — silent grace | ⏳ pending |
| 6 | Release before threshold → tap behavior | ⏳ pending |
| 7 | Conversation continuity (2-turn) | ⏳ pending |
| 8 | Listening state rings: outward, amber, no pulse | ⏳ pending |
| 9 | Dual-utterance color distinction | ⏳ pending |
| 10 | Tap-to-dismiss during conversational audio | ⏳ pending |
| 11 | Telemetry in console | ⏳ pending |
| 12 | Cost telemetry in console | ⏳ pending |
| 13 | Network failure — silent grace | ⏳ pending |
| 14 | Existing tap behavior unchanged | ⏳ pending |

---

## BRANCH AND TAG

| Item | Detail |
|------|--------|
| Branch | `phase-0c-17-conversational` |
| Commit | `89fa25a` — Phase 0c.17: Woadhouse conversational v1 (full pipeline, build 67) |
| Merge to main | ⏳ pending — awaiting device verification |
| Tag v0.6.0-woadhouse-conversational | ⏳ pending |

---

## NOTES FOR NEXT SESSION

**Before first conversational use — seed API keys into device Keychain:**

```swift
// Add temporarily to WallPanelApp.init(), run once, then remove:
KeychainHelper.save(key: "VoiceAPI.anthropicKey", value: "sk-ant-YOUR-KEY-HERE")
KeychainHelper.save(key: "VoiceAPI.cartesiaKey",  value: "sk_car_YOUR-KEY-HERE")
```
Keys persist in Keychain through reinstalls. Remove the seed code after first run.

**Model ID note:** `VoiceAPIConfig.anthropicModel = "claude-haiku-4-5"` — verify against https://docs.anthropic.com before shipping. If wrong, API returns a clear error and the coordinator fails silently.

**Install commands when device is available:**
```bash
APP=/Users/todd/Library/Developer/Xcode/DerivedData/Build/Products/Release-iphoneos/WallPanel.app
xcrun devicectl device install app --device 25423385-294F-50EA-8498-5690FECE21BF "$APP"  # iPad Air
xcrun devicectl device install app --device 8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2 "$APP"  # iPad Pro
```

**After verification passes:**
1. Merge `phase-0c-17-conversational` → `main` (no-ff)
2. Apply tag `v0.6.0-woadhouse-conversational`
3. Update this report with Task 12 results
