# Phase 0c.18 — Streaming pipeline + UI fixes (combined)

> **Task 0 confirmation:** Prompt committed verbatim below before any implementation work began.

---

## Original prompt (verbatim)

```
PHASE 0c.18 — Streaming pipeline + UI fixes (combined)

Three changes in one prompt by Todd's explicit request. Together 
because they touch the same voice subsystem and Todd wants them 
shipped as one thing.

Task 0: Commit this prompt verbatim to the completion report URL 
before any other work. URL at the end.

═══════════════════════════════════════════════════════════════════
SCOPE
═══════════════════════════════════════════════════════════════════

PART A — STREAMING PIPELINE (the big one)

Convert the conversational pipeline from non-streaming end-to-end 
to fully streaming. Target perceived latency: ~750ms from user 
mic-stop to first audible word, vs current ~1100ms.

Three streaming integrations + one bridge:

A1. STREAMING ANTHROPIC RESPONSE
    
    Replace the current /v1/messages call (full response wait) with 
    streaming response via the SSE endpoint. The same /v1/messages 
    endpoint with stream:true returns server-sent events.
    
    - Add stream:true to the request body
    - Parse SSE event stream: content_block_delta events carry 
      partial text in delta.text
    - Accumulate delta.text into a running buffer
    - Emit each token chunk to a downstream consumer (the TTS 
      streamer) as it arrives — do NOT wait for message_stop
    - Continue accumulating until message_stop, then signal 
      end-of-stream to the TTS layer
    
    Architectural change: ClaudeVoiceAssistant.respond() returns an 
    AsyncStream<String> of token chunks instead of a single 
    VoiceAssistantResponse. Or it returns a response object that 
    contains an AsyncStream — your call on the cleanest signature.
    
    The VoiceContext (conversation history, system prompt) 
    construction unchanged.

A2. STREAMING CARTESIA TTS VIA WEBSOCKET
    
    Replace the current Cartesia HTTP API call (full text in, full 
    audio out) with WebSocket streaming. Cartesia exposes a 
    WebSocket endpoint that accepts text chunks and streams audio 
    chunks back as they're synthesized.
    
    Endpoint: wss://api.cartesia.ai/tts/websocket
    Auth: API key as URL parameter or header per Cartesia docs
    Voice config unchanged: sonic-3, Benedict (7cf0e2b1-...), 0.8x 
    speed, neutral, output format raw PCM 16-bit 44.1kHz (or 
    whatever the existing audio pipeline expects).
    
    Critical: Cartesia supports incremental text input. Send text 
    chunks as they arrive from the LLM stream. Cartesia begins 
    synthesizing as soon as it has enough context (typically a 
    sentence boundary or ~10-20 characters).
    
    Audio chunks come back as binary frames. Forward to the audio 
    player as they arrive — do NOT wait for full synthesis.

A3. STREAMING AUDIO PLAYBACK
    
    The current VoiceService.playRawAudio() takes a complete audio 
    buffer. Replace or supplement with a streaming variant: 
    playStreamingAudio(chunks: AsyncStream<Data>) that:
    
    - Sets up AVAudioEngine with a player node and the appropriate 
      format (PCM 16-bit 44.1kHz)
    - Schedules audio buffers as they arrive from the stream
    - Begins playback as soon as the first buffer is queued
    - Continues playback through the stream's natural completion
    - Handles the end-of-stream signal from the Cartesia layer to 
      stop playback when synthesis completes
    
    Existing non-streaming playRawAudio stays available for the 
    bundled phrase pool playback (transitions, etc.) — those don't 
    need streaming.

A4. THINKING-TICK BRIDGE (the perceived-latency win)
    
    The 30 thinking_tick phrases (200-229) bundled in 0c.17a now 
    integrate. They cover the LLM time-to-first-token gap.
    
    Logic in VoiceCoordinator after STT completes:
    
    1. STT produces final transcript
    2. User transcript appears in band (existing behavior)
    3. VoiceForm transitions to .thinking
    4. IMMEDIATELY (parallel, not sequential): pick a random 
       thinking_tick phrase and start playing it via the existing 
       non-streaming playRawAudio path
    5. SIMULTANEOUSLY: fire the streaming Anthropic call
    6. When the first content delta arrives from Anthropic 
       (content_block_delta, not message_start), check if the 
       thinking-tick audio is still playing
       - If still playing: let it complete naturally, then start 
         streaming TTS audio (no crossfade in v1 — clean handoff at 
         tick completion)
       - If already complete: start streaming TTS audio immediately
    7. Cancel/skip the tick if Anthropic responds before tick 
       audio starts (rare but possible)
    
    Decision: NO audio crossfade in this phase. Clean handoff at 
    tick boundary is simpler and avoids audio-engine complexity. 
    If lived use reveals the tick-to-response gap is jarring, 
    crossfade lands in a future fixup.
    
    Ticks selected with no-immediate-repeat (last 8 played). 
    Selection lives in PhraseEventEngine or a new ThinkingTickPicker 
    helper — your call on cleanest placement.

A5. TELEMETRY UPDATES
    
    The existing T1-T10 timestamp framework needs new fields for 
    streaming:
    - T_stream_first_token: first content delta from Anthropic
    - T_tts_first_byte: first audio byte from Cartesia WebSocket
    - T_tick_played: when thinking-tick audio actually started 
      (may be earlier than T_stream_first_token)
    - Perceived latency redefined: min(T_tick_played, T_tts_first_byte) - T1
    
    Console output should make it clear which path the user 
    perceived first (tick vs response).

═══════════════════════════════════════════════════════════════════

PART B — TRANSCRIPT BAND AUTO-SIZE TO CONTENT

Current: TranscriptBand height = HearthStatusBar.height. Long 
Woadhouse responses truncate.

New: band height = max(HearthStatusBar.height, content_height + 
internal_padding). Long responses fully visible.

Constraints:
- Band still anchors absolute bottom, full screen width
- Internal padding preserves vertical centering for short text
- Long responses extend the band upward
- Rotation, background treatment, fade in/out unchanged
- Tap-to-dismiss still works
- Caveat 48pt unchanged
- Word wrapping with appropriate line spacing for Caveat
- Hard max height: 60% of screen height. Beyond that, scroll 
  within band. Expectation is responses won't reach this; if they 
  do, it's a system prompt problem, not a UI problem.

Streaming consideration: as tokens arrive and the response text 
grows, the band must resize gracefully. Smooth height animation 
(150-200ms) when content height changes, OR snap if smooth 
animation conflicts with text appearing. Try smooth first; fall 
back to snap if needed.

═══════════════════════════════════════════════════════════════════

PART C — CLIMATE HERO HORIZONTAL +/- UNDER TARGET TEMP

Current (from 0c.14): +/- buttons inside trinity card adjacent to 
TARGET column.

New: +/- buttons relocated to a horizontal row directly under the 
TARGET temperature number, beneath the "WHAT WE SET" label.

Layout:
- "WHAT WE SET" label at top of TARGET column (existing)
- Target temp number (existing, large Instrument Serif)
- HORIZONTAL +/- button row immediately below the target number
- Buttons sized for comfortable 2-finger touch (~60-70pt circles)
- ~80-100pt centered horizontal spacing between them
- "−" on left, "+" on right
- Existing button treatment (hearthAmberDim border, hearthCream)
- HERE column to the right of TARGET column unaffected

═══════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════

- Single feature branch: phase-0c-18-streaming
- Multiple commits OK within branch — separate logical units (A1, 
  A2, A3, A4, B, C) for traceability if useful
- Tag final commit v0.7.0-streaming
- Clean compile both iPad simulators
- Strict concurrency complete checker remains zero warnings
- Manual verification on iPad if USB cable available; flag if not

REPORT MUST INCLUDE:
- Task 0 confirmation
- All three parts implemented (A streaming, B transcript band, C 
  climate buttons)
- New telemetry baseline numbers if you can capture them in 
  simulator (perceived latency target ~750ms; report what you see)
- Any architectural surprises encountered
- File locations modified
- Whether tick-to-response handoff is clean or has audible gap 
  (lived in simulator)
- Confirmation that bundled phrase pool playback (transitions) 
  still works through the non-streaming path
- API key requirements unchanged: VoiceAPI.anthropicKey and 
  VoiceAPI.cartesiaKey both still required and read from Keychain

COMPLETION REPORT AT:
https://github.com/tlavigne974os/home-control-handoff/blob/main/completions/2026-04-28-phase-0c-18-streaming.md
```

---

## Implementation status

> *This section will be filled in as work completes.*

### Task 0 ✅
Prompt committed verbatim. Branch `phase-0c-18-streaming` created.

### Part A — Streaming pipeline
*In progress*

### Part B — TranscriptBand auto-size
*In progress*

### Part C — Climate hero +/- relocation
*In progress*

---

*Report will be finalized after all parts complete.*
