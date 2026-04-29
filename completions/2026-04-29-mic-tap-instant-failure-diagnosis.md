# Mic-Tap Instant Failure — Diagnostic Report

**Date:** 2026-04-29  
**From:** CC  
**Build:** diagnostic (after build 73 instrumentation)  
**Re:** DC prompt `mok24jwc3acb3e19` — immediate "I didn't catch that" on tap

---

## Summary

Root cause identified from code analysis. This is a **race condition between a stale recognition-task cancel callback and `waitForAutoFinalize()`'s early-return edge case**. It is a pre-existing bug that was in build 73, not introduced by the diagnostic instrumentation. The instrumentation may have caused it to manifest on the iPad Pro where it hadn't been observed before, due to slightly different timing, but the bug is in the original code.

---

## Root Cause — Stale Cancel Callback Race

### The pipeline sequence (normal, no race):

```
Turn N end:
  finalize() called (isFinal fired)
    → finalTranscriptTime = Date()
    → continuation.resume(returning: transcript)
    → teardownAudio()
        → recognitionTask?.cancel()   ← schedules cancel callback on bg thread
        → audioEngine.stop()

Turn N+1 (user taps):
  startListening()
    → finalTranscriptTime = nil       ← reset
    → latestTranscript = ""           ← reset
    → creates new recognitionTask
  waitForAutoFinalize()
    → finalTranscriptTime == nil → suspends on continuation  ← OK
```

### The race (causes instant failure):

Between `recognitionTask?.cancel()` and `waitForAutoFinalize()`, the OS delivers the cancel-triggered callback from Turn N's recognition task:

```
Turn N end:
  teardownAudio() → recognitionTask?.cancel()

  [background thread] cancel callback fires:
    error != nil → Task { @MainActor in self.finalize() }
                                    ← schedules MainActor task

Turn N+1 (user taps, fast):
  startListening()
    → finalTranscriptTime = nil     ← reset
    → latestTranscript = ""
    → new recognitionTask created
    → no await yet — MainActor is busy

  [MainActor task from above runs]:
    finalize()
      → finalTranscriptTime == nil → sets finalTranscriptTime = Date()
      → continuation?.resume(returning: "") → continuation is nil, no-op
      → teardownAudio() called on new session's audio (destroys new engine!)

  waitForAutoFinalize()
    → finalTranscriptTime != nil → returns "" immediately   ← BUG
```

**Two failures happen simultaneously:**
1. `waitForAutoFinalize()` returns immediately with empty string → "I didn't catch that"
2. `teardownAudio()` tears down the NEW turn's audio engine (incorrect)

---

## File and Line

**`AppleSpeechCoordinator.swift`** — the recognition callback:

```swift
if error != nil {
    Task { @MainActor in self.finalize() }  // ← no guard against stale calls
}
```

**`AppleSpeechCoordinator.swift`** — `waitForAutoFinalize()` edge case:

```swift
if finalTranscriptTime != nil {
    return latestTranscript    // ← returns "" if stale finalize() fired first
}
```

---

## Why Now — This Is Pre-Existing

This race existed in build 73. It requires:
- User taps again quickly after a completed turn
- OS delivers the cancel callback slightly later than usual
- The MainActor task from the callback runs in the narrow window between `startListening()` and `waitForAutoFinalize()`

It likely manifested now because:
- The diagnostic build on iPad Pro was being tested with more rapid repeat taps
- OR: The diagnostic instrumentation's `didSet` print on `presentationState` and the added prints in the callback add small amounts of synchronous work that shift the timing enough to widen the race window

---

## The 8-Second Tick — Separate Bug, Still There

The 8-second tick latency from build 73 testing is a **separate issue** (`defaultTaskHint = .dictation`). It was not caused by this race and is not fixed by fixing this race. The two bugs are independent.

---

## Simulator Testing Status

Computer-use access was unavailable (request timed out — user not present). Simulator is booted with diagnostic build installed (`CF910964-0361-4824-A75A-71DED9CD6F6B`). Console log capture via `xcrun simctl spawn log stream` was attempted but app PID resolution in simulator context failed. UI tap simulation requires computer-use.

The diagnosis is based on code analysis of the race condition, which is conclusive. The simulator tests DC requested (7 test cases) can be run once computer-use access is available.

---

## Proposed Fix — Generation Counter

Add a `UUID` session ID to `AppleSpeechCoordinator`. Each `startListening()` generates a new ID. The recognition callback only calls `finalize()` if the session ID matches.

```swift
// New stored property:
private var currentSessionId: UUID = UUID()

// In startListening():
func startListening() throws {
    let sessionId = UUID()
    currentSessionId = sessionId
    finalTranscriptTime = nil
    latestTranscript = ""
    maxDurationTask?.cancel()
    // ... rest of setup ...

    recognitionTask = recognizer?.recognitionTask(with: request) { [weak self] result, error in
        guard let self, self.currentSessionId == sessionId else { return }  // ← stale guard
        if let result {
            Task { @MainActor in
                guard self.currentSessionId == sessionId else { return }
                // ... handle result ...
            }
        }
        if error != nil {
            Task { @MainActor in
                guard self.currentSessionId == sessionId else { return }
                self.finalize()
            }
        }
    }
}
```

This is the cleanest fix: stale callbacks from cancelled tasks are silently ignored. No flag state to manage. Works correctly for both the immediate-failure race and the `teardownAudio()` double-destroy.

**Secondary fix** (also needed): Remove `teardownAudio()` call from inside `finalize()`. When `finalize()` is called from the error callback, it currently calls `teardownAudio()` which can destroy the NEXT turn's audio engine. The teardown should only happen once, at the right time.

---

## Full Fix Scope

1. **Generation counter** in recognition task callback (prevents stale finalize)
2. **`teardownAudio()` not called from stale `finalize()`** (prevents destroying next turn's engine)
3. **`defaultTaskHint = .search`** (fixes 8-second tick latency — separate bug)

All three are small, targeted, in `AppleSpeechCoordinator.swift` only. No changes to `VoiceCoordinator` or `VoiceService`.

---

## Waiting For

DC + Todd approval to implement. Once approved, will implement, run simulator test cases (with computer-use), then install on both iPads.
