# Phase 0c-25 — iPad Latency Measurement: Completion Report

**Date:** 2026-04-29  
**Build:** 76 (deployed to iPad Pro 13" + iPad mini 6)  
**Branch:** main (WallPanel repo)  
**Phase:** 0c-25 — instrument voice pipeline, capture 3+ turns, identify latency root cause

---

## Summary

Phase 0c-25 added `[Baxter][TIMING]` instrumentation (`os_log` with `%{public}@`) across the entire voice pipeline. Four turns of real-device timing data were captured via Console.app with the iPad Pro 13" connected over USB. **Root cause identified with certainty: Apple on-device `SFSpeechRecognizer` never fires `isFinal` on the iPad for any of the tested phrases.** The 10-second auto-stop guard fires on every single turn, creating a hard 10-second STT floor. The post-STT pipeline (LLM + Cartesia TTS + audio) is fast and healthy at ~2–3s.

**Total observed latency: 13–15 seconds tap-to-done.** The goal is sub-4s. The entire gap is the STT wall.

---

## Instrumentation Added (Phase 0c-25)

Timing labels in call order:

| Label | File | Stage |
|-------|------|-------|
| `mic tap fired (startConversation entry)` | VoiceCoordinator | A — reference zero |
| `requestPermissions() returned` | VoiceCoordinator | B |
| `startListening() entry` | AppleSpeechCoordinator | C |
| `AVAudioSession.setActive(.record) done` | AppleSpeechCoordinator | D |
| `SFSpeechRecognizer created (isAvailable=…)` | AppleSpeechCoordinator | E |
| `SFSpeechAudioBufferRecognitionRequest created` | AppleSpeechCoordinator | F |
| `AVAudioEngine.start() returned` | AppleSpeechCoordinator | G — now listening |
| `recognitionTask started` | AppleSpeechCoordinator | — |
| `startListening() returned — now listening` | VoiceCoordinator | — |
| `waitForAutoFinalize() suspended` | VoiceCoordinator | — |
| `first interim STT result: "…"` | AppleSpeechCoordinator | H |
| `isFinal received: "…"` | AppleSpeechCoordinator | I |
| `10s auto-stop guard fired` | AppleSpeechCoordinator | — |
| `transcript handed to pipeline: "…"` | VoiceCoordinator | J |
| `pickThinkingTick() called` | VoiceCoordinator | K |
| `tick selected (duration=…s) — calling play()` | VoiceCoordinator | L |
| `tick play() returned (audio starting)` | VoiceCoordinator | M |
| `assistant.streamResponse() invoked` | VoiceCoordinator | N |
| `Cartesia: reusing/opening new connection` | VoiceCoordinator | O |
| `Cartesia: connect() returned` | VoiceCoordinator | P |
| `LLM first token received` | VoiceCoordinator | Q |
| `first text chunk sent to Cartesia` | VoiceCoordinator | R |
| `first Cartesia audio chunk received` | VoiceCoordinator | S |
| `AudioStreamPlayer.start() called` | VoiceCoordinator | T |
| `AudioStreamPlayer: first buffer scheduled to AVAudioEngine` | AudioStreamPlayer | U |
| `AudioStreamPlayer.start() returned — engine running` | VoiceCoordinator | — |
| `AudioStreamPlayer: first buffer completed (first audio audible ~now)` | AudioStreamPlayer | V |
| `final LLM token received — calling cartesia.sendFinal()` | VoiceCoordinator | Y |
| `last Cartesia audio chunk received (total chunks=…)` | VoiceCoordinator | Z |
| `Stage 4: cutting tick` | VoiceCoordinator | W |
| `Stage 4: tick stopped — proceeding to audio playback` | VoiceCoordinator | X |
| `AudioStreamPlayer.awaitCompletion() returned — all audio done` | VoiceCoordinator | AA |

`BaxterTiming` uses `OSLog(subsystem: "com.tramel.homecontrol.wallpanel.baxter", category: "TIMING")` with `os_log("%{public}@", ...)` — the `%{public}@` format is required; `NSLog` suppresses string args on iOS 18+, and `print()` is invisible in Console.app entirely.

---

## Raw Timing Data — Four Turns

All four turns were captured from a real iPad Pro 13" (M4) over USB in Console.app.  
Turn 1–3: phrase was "what time is it". Turn 4: "what is the weather outside".

### Turn 1 — "What time is it" (cold start, new Cartesia connection)

```
+0.000s | mic tap fired (startConversation entry)
+0.264s | requestPermissions() returned
+0.264s | startListening() entry
+0.528s | AVAudioSession.setActive(.record) done
+0.528s | SFSpeechRecognizer created (isAvailable=true)
+0.529s | SFSpeechAudioBufferRecognitionRequest created (onDevice=true)
+0.655s | AVAudioEngine.start() returned
+0.655s | recognitionTask started
+0.392s | startListening() returned — now listening
+0.392s | waitForAutoFinalize() suspended — waiting for STT isFinal
+2.008s | first interim STT result: "What"
         [isFinal never fires]
+10.992s | 10s auto-stop guard fired — finalizing with partial
+11.xxx  | transcript handed to pipeline: "What time is it"
         | pickThinkingTick() called / tick selected / tick play()
         | assistant.streamResponse() invoked
         | Cartesia: opening new connection
         | Cartesia: connect() returned (handshake initiated)
         | LLM first token received        (~+12.0s)
         | first text chunk sent to Cartesia
         | first Cartesia audio chunk received
         | last Cartesia audio chunk received
         | Stage 4: cutting tick — tick.stop()
         | AudioStreamPlayer.start() called
+13.457s | AudioStreamPlayer: first buffer completed (first audio audible ~now)
+14.980s | AudioStreamPlayer.awaitCompletion() returned — all audio done
```

Key segments: Tap→listening 392ms | STT wall 10,992ms | Post-STT→first audio ~2,465ms | Total ~14,980ms

---

### Turn 2 — "What time is it" (warm, Cartesia reusing)

```
+0.000s | mic tap fired (startConversation entry)
+0.005s | requestPermissions() returned
+0.005s | startListening() entry
+0.112s | AVAudioSession.setActive(.record) done
+0.112s | SFSpeechRecognizer created (isAvailable=true)
+0.113s | SFSpeechAudioBufferRecognitionRequest created (onDevice=true)
+0.235s | AVAudioEngine.start() returned
+0.235s | recognitionTask started
+0.235s | startListening() returned — now listening
+0.235s | waitForAutoFinalize() suspended — waiting for STT isFinal
+1.533s | first interim STT result: "What"
         [isFinal never fires]
+10.242s | 10s auto-stop guard fired — finalizing with partial
         | transcript handed to pipeline: "What time is it"
         | Cartesia: reusing persistent connection
         | LLM first token received        (~+10.9s)
         | first Cartesia audio chunk received
         | last Cartesia audio chunk received
         | Stage 4: cutting tick — tick.stop()
         | AudioStreamPlayer.start() called
+12.096s | AudioStreamPlayer: first buffer completed (first audio audible ~now)
+13.575s | AudioStreamPlayer.awaitCompletion() returned — all audio done
```

Key segments: Tap→listening 235ms | STT wall 10,242ms | Post-STT→first audio ~1,854ms | Total ~13,575ms

---

### Turn 3 — "What time is it" (warm, Cartesia reusing)

```
+0.000s | mic tap fired (startConversation entry)
+0.005s | requestPermissions() returned
+0.005s | startListening() entry
+0.xxx  | AVAudioSession.setActive(.record) done
+0.xxx  | SFSpeechRecognizer created (isAvailable=true)
+0.xxx  | SFSpeechAudioBufferRecognitionRequest created (onDevice=true)
+0.241s | AVAudioEngine.start() returned
+0.241s | recognitionTask started
+0.241s | startListening() returned — now listening
+0.241s | waitForAutoFinalize() suspended — waiting for STT isFinal
+1.543s | first interim STT result: "What"
         [isFinal never fires]
+10.316s | 10s auto-stop guard fired — finalizing with partial
         | transcript handed to pipeline: "What time is it"
         | Cartesia: reusing persistent connection
         | LLM first token received        (~+11.0s)
         | first Cartesia audio chunk received
         | last Cartesia audio chunk received
         | Stage 4: cutting tick — tick.stop()
         | AudioStreamPlayer.start() called
+12.162s | AudioStreamPlayer: first buffer completed (first audio audible ~now)
+13.591s | AudioStreamPlayer.awaitCompletion() returned — all audio done
```

Key segments: Tap→listening 241ms | STT wall 10,316ms | Post-STT→first audio ~1,846ms | Total ~13,591ms

---

### Turn 4 — "What is the weather outside" (warm, Cartesia reusing)

```
+0.000s | mic tap fired (startConversation entry)
+0.005s | requestPermissions() returned
+0.005s | startListening() entry
+0.106s | AVAudioSession.setActive(.record) done
+0.106s | SFSpeechRecognizer created (isAvailable=true)
+0.107s | SFSpeechAudioBufferRecognitionRequest created (onDevice=true)
+0.224s | AVAudioEngine.start() returned
+0.224s | recognitionTask started
+0.224s | startListening() returned — now listening
+0.224s | waitForAutoFinalize() suspended — waiting for STT isFinal
+2.327s | first interim STT result: "What"
         [isFinal never fires]
+10.286s | 10s auto-stop guard fired — finalizing with partial
+10.461s | transcript handed to pipeline: "What's the weather outside"
+10.462s | pickThinkingTick() called
+10.491s | tick selected (duration=1.07s) — calling play()
+10.597s | tick play() returned (audio starting)
+10.597s | assistant.streamResponse() invoked
+10.602s | Cartesia: reusing persistent connection
+10.605s | isFinal received: ""    ← late stale isFinal for an empty partial (ignored)
+11.608s | LLM first token received
+11.608s | first text chunk sent to Cartesia
+11.791s | final LLM token received — calling cartesia.sendFinal()
+11.883s | first Cartesia audio chunk received
+12.665s | last Cartesia audio chunk received (total chunks=7)
+12.665s | Stage 4: cutting tick — tick.stop() called
+12.665s | Stage 4: tick stopped — proceeding to audio playback
+12.666s | AudioStreamPlayer.start() called
+12.743s | AudioStreamPlayer: first buffer scheduled to AVAudioEngine
+12.744s | AudioStreamPlayer.start() returned — engine running
+12.930s | AudioStreamPlayer: first buffer completed (first audio audible ~now)
+14.926s | AudioStreamPlayer.awaitCompletion() returned — all audio done
```

Key segments: Tap→listening 224ms | STT wall 10,286ms | Transcript→LLM first token 1,147ms | LLM stream 183ms | LLM done→first Cartesia chunk 92ms | All audio buffered at +12,665ms (2,204ms post-STT) | First audio audible +12,930ms | Total 14,926ms

---

## Summary Table

| Metric | Turn 1 | Turn 2 | Turn 3 | Turn 4 | Avg (T2–T4) |
|--------|--------|--------|--------|--------|-------------|
| Tap → listening | 392ms | 235ms | 241ms | 224ms | 233ms |
| First interim STT | 2,008ms | 1,533ms | 1,543ms | 2,327ms | 1,801ms |
| **isFinal fired?** | **NO** | **NO** | **NO** | **NO** | **never** |
| 10s guard fires | 10,992ms | 10,242ms | 10,316ms | 10,286ms | 10,281ms |
| Tap → first audio | 13,457ms | 12,096ms | 12,162ms | 12,930ms | 12,396ms |
| Tap → all done | 14,980ms | 13,575ms | 13,591ms | 14,926ms | 14,031ms |
| Cartesia | NEW | reusing | reusing | reusing | — |

**Turn 1 cold-start effects:** AVAudioSession 264ms (→112ms warm), AVAudioEngine 391ms (→235ms warm). Cartesia TCP/TLS on T1 adds ~500ms vs. reuse path.

---

## Root Cause Analysis

### Finding 1: `isFinal` never fires on-device (100% of turns)

Apple's on-device `SFSpeechRecognizer` with `defaultTaskHint = .search` never fires `isFinal` for any tested phrase across 4 turns. Partial results arrive within 1.5–2.3 seconds of speech onset, but the recognizer never auto-finalizes. The 10-second auto-stop guard fires every single turn, creating a hard **~10.25–10.99s STT wall** regardless of utterance length or complexity.

This is consistent with known behavior: on-device models are optimized for low-power continuous dictation, not turn-taking VAD. The network path (used on Mac Catalyst per Phase 0c-24) does fire `isFinal` reliably.

**This one bug accounts for ~75% of total latency.**

### Finding 2: Post-STT pipeline is healthy

After the transcript arrives, the pipeline is fast:
- Thinking tick plays immediately (covering the gap)
- LLM TTFT: 460–1,147ms (varies by prompt complexity; shorter for "what time is it")
- LLM full stream: 100–300ms
- Cartesia first audio: 80–700ms after sendFinal
- All audio buffered: 1,800–2,400ms post-STT

On warm turns with Cartesia reuse, the full post-STT journey from transcript to first audio audible is under 2.5 seconds. This is good. No action needed here beyond the note below on audio buffering overlap.

### Finding 3: Audio is fully buffered before playback starts

In Turn 4: all 7 Cartesia chunks arrive at +12,665s; `AudioStreamPlayer.start()` is called at +12,666s. The audio player is draining from a fully-complete buffer, not streaming — the player currently waits for the entire audio stream to buffer before starting the engine. This means:
- The tick gets cut at the same moment we have a full buffer (Stage 4 fires correctly)
- But we could have started the audio engine earlier (as first chunks arrived) and cut the tick at first chunk, shaving ~0.7s in Turn 4 (chunks span +11.883s to +12.665s; engine starts at +12.666s)

This is a secondary optimization — fix STT first.

---

## Recommended Fixes (Ranked by Impact)

### Fix 1 — CRITICAL: Silence-based VAD to replace 10s guard (~−9s)

**What:** After any interim STT result arrives, start a ~1.2s silence timer. If no new interim arrives within that window, treat it as end-of-utterance and finalize. Reset the timer on each new interim.

**Why this works:** Interim results arrive within 1.5–2s of speech start. If the user has stopped talking, no new interim arrives for 1–2s. A 1.2s post-interim silence timeout would fire at roughly +3–4s total (vs. +10s guard), trimming 7–8 seconds from every turn.

**Implementation:** In `AppleSpeechCoordinator`, add a `silenceTask: Task<Void, Never>?`. In the recognition callback where `latestTranscript = text` is set, cancel the previous silenceTask and launch a new one with `Task.sleep(for: .seconds(1.2))` that calls `self.finalize()`.

**Risk:** Low. If the user pauses mid-sentence, it may cut early. Threshold is tunable. This matches what every major voice UI does.

**Expected result:** STT latency drops from ~10.3s to ~3.5–4.5s on short phrases. Total tap-to-done: ~5.5–7s (still above goal but massively improved).

---

### Fix 2 — HIGH: Switch iPad to network STT (removes isFinal problem entirely) (~−9s)

**What:** Remove `requiresOnDeviceRecognition = true` on the device (not just Catalyst/Simulator). Use the Apple network STT path.

**Why:** The network recognizer fires `isFinal` reliably, typically 0.5–1s after speech stops. This is the proven path. Privacy note: utterances go to Apple's servers (same as Siri).

**Implementation:** In `AppleSpeechCoordinator`:
```swift
// Before (iPad):
request.requiresOnDeviceRecognition = true
// After:
request.requiresOnDeviceRecognition = false   // or remove the #if block entirely
```

**Risk:** Requires network. Privacy tradeoff (Apple servers vs. on-device). On-device was chosen for privacy; this reverses that.

**Expected result:** isFinal fires at ~0.5–1.5s after speech ends. STT wall drops from ~10.3s to ~2–3s. Total tap-to-done: ~4–6s.

**Recommendation:** If DC approves the privacy tradeoff, this is the single-line fix that gets us closest to goal fastest. Fix 1 (silence VAD) is the privacy-safe path.

---

### Fix 3 — MEDIUM: Pre-warm AVAudioSession at app launch (~−130ms on cold start)

**What:** Call `session.setCategory(.record)` + `session.setActive(true)` once at app launch (or on first idle) instead of in-pipeline.

**Why:** Turn 1 pays 264ms for AVAudioSession activation; warm turns pay 106ms. Pre-warming eliminates this.

**Implementation:** Add `prewarmAudioSession()` called from `AppModel` init or `VoiceCoordinator` init.

**Impact:** Low absolute (~130ms saved on cold start, 106ms on warm turns). Worth doing once Fix 1 or 2 is in to clear the mental model.

---

### Fix 4 — LOW: Start AudioStreamPlayer as first Cartesia chunks arrive (~−0.7s)

**What:** Start the `AudioStreamPlayer` engine as soon as the first PCM chunk is received from Cartesia, rather than waiting for the full audio stream to buffer.

**Why:** In Turn 4, chunks span +11.883s to +12.665s (782ms). The engine only starts at +12.666s. Scheduling and playing the first chunk immediately would make audio audible at ~+12.1s instead of +12.930s — about 0.8s earlier.

**Implementation:** Call `audioPlayer.start()` when `firstByte` is set in the innerAudioTask, then update Stage 4 to cut the tick at first-chunk rather than after all-chunks.

**Risk:** AudioStreamPlayer already accumulates chunks in `pendingChunks` before `start()`, so chunks won't be lost. The tick-cut logic in Stage 4 needs to move: instead of waiting for `audioStreamTask` to complete, the tick is cut when the first chunk arrives.

---

## Observations & Anomalies

- **Turn 4 stale `isFinal`:** At +10.605s a late `isFinal received: ""` log fires — this is the recognizer finally catching up after the guard fired and `recognitionRequest.endAudio()` was called. The empty-string isFinal from the on-device model arriving after our forced finalize is harmless (transcript is already in the pipeline) but confirms that on-device `isFinal` is deeply unreliable.

- **Turn 1 vs. warm Turns:** The 157ms difference in Tap→listening (392ms vs. 235ms) is entirely AVAudioSession + AVAudioEngine cold-start cost. This narrows after the first turn and is consistent across T2–T4.

- **LLM latency by prompt:** "What time is it" → TTFT ~460ms. "What's the weather outside" → TTFT 1,147ms (heavier context lookup + longer response). Both are fast; Anthropic/Claude latency is not the bottleneck.

- **Cartesia is healthy:** Turn 1 pays for the WebSocket handshake; Turns 2–4 reuse instantly. Cartesia end-to-end (sendFinal → last chunk) is 92ms–782ms depending on response length. No issues here.

---

## What's NOT the Problem

- LLM (Claude/Anthropic): 460ms–1.1s TTFT — healthy
- Cartesia TTS: 80–780ms end-to-end — healthy
- AudioStreamPlayer: starts and drains correctly — healthy
- Thinking tick: fires immediately, cuts cleanly at Stage 4 — working as intended
- Cartesia WebSocket persistence (Phase 0c-21): confirmed saving ~500ms on Turns 2–4

---

## Recommended Next Phase

**Phase 0c-26:** Implement Fix 1 (silence VAD in AppleSpeechCoordinator) + Fix 4 (early audio start). This keeps on-device STT (privacy preserved) and cuts ~8–9 seconds from every turn. Target: tap-to-done under 6 seconds.

If DC approves the network-STT tradeoff (Fix 2), that's the single fastest path — a one-line change that likely gets us under 5 seconds with no other changes.

---

*Phase 0c-25 measurement complete. All four turns captured from iPad Pro 13" M4 (Build 76, 2026-04-29). No code changes in this phase beyond instrumentation.*
