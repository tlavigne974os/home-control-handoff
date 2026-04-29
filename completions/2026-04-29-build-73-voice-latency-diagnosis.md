# Voice Latency Diagnosis — Build 73

**Date:** 2026-04-29  
**From:** CC  
**Re:** DC prompt `mok1n9km679d73ba` — voice regression, 8s tick / 11s audio

---

## Todd's Question: "Is it all local? Should be faster?"

**Yes — entirely local. Yes — should be significantly faster.**

`AppleSpeechCoordinator` line 79: `request.requiresOnDeviceRecognition = true`

There are zero network calls in the STT path. Everything happens on the Neural Engine. The 8 seconds is on-device latency.

---

## Root Cause: `defaultTaskHint = .dictation`

**File:** `WallPanel/WallPanel/Services/Voice/AppleSpeechCoordinator.swift`, line 75

```swift
newRecognizer?.defaultTaskHint = .dictation
```

`SFSpeechRecognitionTaskHint.dictation` is tuned for long-form document dictation — it uses a silence detection window of approximately **2–3 seconds** after the last word before firing `isFinal`. This is intentional for dictation: you pause between sentences and the recognizer should wait before finalizing.

For voice assistant queries ("Do you know what version you are?"), we need `.search`, which fires `isFinal` within **~0.5–1 second** of silence after the last word.

This is the primary contributor to the 8-second delay.

---

## Pipeline Anatomy — Where the 8 Seconds Live

The tick fires in Stage 2, immediately after `waitForAutoFinalize()` returns:

```
T1  user taps mic
     ↓ (synchronous)
T2  startListening() called — AVAudioEngine starts, recognizer starts
     ↓ ~0.5–1.5s: audio session setup, engine.prepare(), engine.start()
     ↓            recognizer initialization, first audio buffer flowing
T3  first partial result
     ↓ ~2.5–3s: dictation silence window elapses post-last-word
T4  isFinal fires → waitForAutoFinalize() returns
     ↓ (synchronous — 0ms)
     tick plays ← THIS IS WHAT TODD HEARS AT ~8s
```

Estimated breakdown of ~8 seconds:
- Audio session deactivate (.playback) → activate (.record): ~0.5s
- AVAudioEngine prepare/start: ~0.5–1s  
- Recognizer startup, first audio buffer: ~0.5–1s
- User speaking "Do you know what version you are?": ~3s
- `.dictation` post-speech silence window before isFinal: ~2.5–3s
- **Total: ~7–9s ✓**

With `.search`, the post-speech silence window drops to ~0.5–1s:
- Same pipeline minus dictation silence: **~4.5–6s** → tick at ~5s not 8s

---

## Suspects Ranked

**1. `defaultTaskHint = .dictation` — PRIMARY CULPRIT** (~2–3s penalty)  
One-line fix: `.dictation` → `.search`

**2. Per-turn recognizer initialization** (~1–1.5s overhead)  
`SFSpeechRecognizer`, `AVAudioEngine`, `SFSpeechAudioBufferRecognitionRequest` all created fresh on every tap. The recognizer has a warm-up phase where early audio may not be processed. Mitigation: pre-initialize the recognizer once; create only the request per turn.

**3. Audio session category switching** (~0.5s)  
Previous turn ends: `setCategory(.playback)` + `setActive(true)`.  
Next tap: `setCategory(.record)` + `setActive(true)`.  
The deactivate/activate roundtrip adds latency. Mitigation: keep session active in ambient record mode between turns (more complex, deferred).

**4. Persistent CartesiaWSClient (DC suspect #1) — NOT the tick culprit**  
The Cartesia connection is only used in Stage 3, which starts AFTER the tick fires. Stale Cartesia could add latency to the 8s→11s phase (LLM+TTS) but does not affect the tick at 8s. The `isConnected` check should be verified separately but it's not the tick issue.

**5. Stage 4 tick gate (DC suspect #2) — NOT the tick culprit**  
`tickPlayer?.stop()` fires in Stage 4, after LLM+TTS. The tick starting at 8s is Stage 2. Part 3 is irrelevant to tick latency.

---

## Is This a Regression From 0c.21?

**Unclear — may be confirmation bias.**

The `defaultTaskHint = .dictation` and per-turn initialization code was not changed in 0c.21. `AppleSpeechCoordinator` was untouched. The STT latency was likely the same in builds 70/71.

What 0c.21 changed:
- Stage 4 tick gate: previously the pipeline awaited tick completion before starting audio (added ~2s post-tick wait). Build 73 cuts the tick immediately → audio starts faster.
- The overall tap-to-audio-complete time is likely SHORTER in build 73 than 70/71.

The perceived regression may be that in 70/71, the tick played for its full duration (giving 2+ seconds of audible "I'm thinking" feedback after 8s), whereas in build 73 the tick gets cut short quickly. The **subjective** experience of "Baxter is working on it" may have degraded even if the total latency improved. This is a UX perception issue worth investigating separately.

---

## Diagnostic Build Installed

Build with timing prints deployed to iPad Pro (`8B58895C`). iPad Air (`25423385`) was unreachable at install time (tunnel timeout — sleeping). The timing prints will emit to Console.app filtered on `[Baxter] ⏱`:

```
[Baxter] ⏱ T2 STT started — <timestamp>
[Baxter] ⏱ T3 STT first partial — X.XXs — "first words"
[Baxter] ⏱ isFinal fired — taskHint=dictation
[Baxter] ⏱ T4 STT finalized — X.XXs — "full transcript"
```

Plus the existing `[VoiceTelemetry]` block at turn end showing `stt:` duration.

---

## Proposed Fix — Pending DC + Todd Approval

**Phase 0c.22 / Fix A (one line — immediate impact):**
```swift
// AppleSpeechCoordinator.swift line 75
newRecognizer?.defaultTaskHint = .search  // was .dictation
```
Expected: tick latency drops from ~8s to ~5s. No other changes.

**Phase 0c.22 / Fix B (moderate — pre-warm recognizer):**
Initialize `SFSpeechRecognizer` once at startup (not per-turn). Only create the request + engine per turn. Expected: additional ~0.5–1s reduction.

**Phase 0c.22 / Fix C (UX — restore tick duration perception):**
If the tick-getting-cut-short is why 73 feels worse subjectively, consider a minimum tick duration (e.g., 1s) before cutting. The tick's job is to give audible feedback that Baxter heard you — cutting it at 0.1s defeats that purpose even if audio starts "sooner."

---

## Recommendation

Do NOT revert Parts 3 or 4 of 0c.21. They are not the cause. A targeted one-line `.dictation` → `.search` fix is the right first move. Validate with Todd (tick at ~5s feels materially better than 8s), then address Fix B as a follow-on.

Waiting for DC + Todd sign-off before implementing.
