# Home Control Center — Technical Architecture Audit (TAuC)

**Date:** 2026-04-28
**Auditor:** TAuC — Technical Architectural Audit by Claude (fresh-context Opus)
**Build reviewed:** 71 (compiled, on iPad: 70)
**Scope:** Every major architectural decision and every major subsystem. Code-level evidence with file paths and line numbers. Out of scope: code health (CC), strategic narrative (DC), team-structure (AUC).
**Materials walked:** brief; three prior audits; entire voice subsystem (12 files, ~2,000 LOC); HAClient + PollingHAClient + HAController + AppModel + PanelStateController + PhotoController; AmbientView/RootView/MainPanelView; HomeControlCore (Models, Design tokens, PhotoLibrary, PanelConfig); deploy.sh; tools/render-phrases.py; Package.swift; pbxproj concurrency settings; live `xcrun devicectl list devices`.

---

## Executive summary

The architecture is mostly sound, but it has more dead and half-wired code than the project narrative acknowledges, and the boundary between Layer 1 and Layer 2 is less clean than the brief implies. The HAClient actor pattern, the `VoiceAssistant` protocol, and `PanelConfig` will all extend cleanly into 0d.2; `HAController` write stubs are in the right places and typed correctly. The single biggest pre-0d.2 risk isn't writes themselves — it's the parallel audio pipelines, the dead `respond()` / `playRawAudio()` / `CartesiaTTSClient` paths, and the `[String: any Sendable]` payload typing in `callService` that needs to be revisited before writes go live. Strict concurrency is enabled only on the core package, not the app target, and most of the voice subsystem lives in the unchecked half.

---

## Section A — Core architectural decisions

### A.1 — SwiftUI as the UI framework

**Verdict: Sound but with caveats.**

iPadOS 17 baseline + SwiftUI Observation framework is the right call for an always-on full-screen wall panel. The `@Observable` macro avoids the ObservableObject ceremony, and transitive observation through computed properties (AppModel forwarding from HAController/PhotoController/PanelStateController) works correctly in 17+. Animation primitives (`withAnimation`, `.transition`, `Animation.spring`) are doing the bulk of the project's "Living motion" discipline cheaply.

Caveats visible in code:
- `MainPanelView.swift:221` reads `UIScreen.main.bounds.width` — deprecated in iPadOS 17 multi-window/Stage Manager contexts and should be sourced from `GeometryReader` already enclosing the parent. On a wall panel that never enters Stage Manager this is functionally fine; it's a code-smell that will produce a warning eventually.
- Photo loading (`PhotoController.advancePhoto`) is async and writes `currentImage` on `MainActor` — animations span an `await`, so the crossfade can land mid-rotation if the user opens quick control during the gap. Not user-visible enough to fix, but worth knowing.
- The `@MainActor`-everywhere convention (every controller, AppModel, all voice classes) reduces concurrency reasoning surface but means every async pipeline touches the main thread. With 0d.2 writes adding more network calls, this stays fine — Swift Concurrency hops cheaply. With 0d.3 WebSocket events arriving at high frequency, watch for main-thread contention.

When 0d.2 wires writes: SwiftUI will not strain. Optimistic-update writes that mutate `home` inside `withAnimation` are already the established pattern (see `applySnapshot` in `HAController.swift:122–129`). Adding eight more such mutations is mechanical.

### A.2 — The HAClient actor pattern

**Verdict: Sound.**

The protocol/actor split (`HomeControlCore/Sources/HomeControlCore/Client/HAClient.swift`) is exactly the right shape for what this is doing. `HAClient` is declared as `protocol HAClient: Actor`, which gates conformers to actor types. `PollingHAClient` is the concrete actor; `HAController` is `@MainActor`; cross-boundary calls are explicit `await` (`HAController.swift:112`, `116`). Sendable boundaries are clean: `HomeSnapshot`, `EntityHealth`, `StateChangeEvent` are all Sendable structs.

The pattern extends cleanly to 0d.2. `callService(domain:service:data:)` is already declared on the protocol with a `[String: any Sendable]` parameter (line 61); the stub returns `notImplemented` and the eight write methods on `HAController` mirror this stub-and-throw shape. Adding write bodies is a contained change — the protocol surface, the controller method signatures, and the optimistic-update pattern are all in place.

One real concern at the boundary: `[String: any Sendable]` is the typing escape hatch the architectural prep phase explicitly logged as "to be revisited in 0d.2." It compiles, but it gives back none of the type safety the actor migration nominally bought. Concretely, you can pass `["temperature": NSNumber(...)]` and Swift won't catch the bridging error until runtime. Replace with a typed `Encodable` payload wrapper before the first write ships. This is the single architectural decision that needs to be made *before* 0d.2 starts, not during it — once write call sites have learned the bag-of-Sendable shape, it's harder to walk back.

I checked for actor-isolation defeats: no `@unchecked Sendable`, no `nonisolated(unsafe)`, no isolated escapes. The only `nonisolated` annotations are on `AVAudioPlayerDelegate.audioPlayerDidFinishPlaying` (`VoiceService.swift:465`) and `PHPhotoLibraryChangeObserver.photoLibraryDidChange` (`PhotoLibrary.swift:151`) — both required by Apple protocols and both correctly marshal to `@MainActor` via `Task { @MainActor in … }`. Clean.

### A.3 — Polling instead of WebSocket

**Verdict: Sound but with caveats.**

10-second REST polling is the right MVP cadence. `PollingHAClient.fetchAllStates` uses `timeoutInterval: 10` (line 101), which means a 10-second poll can effectively become a 20-second gap if the previous fetch is timing out — `HAController.fetchHA` awaits the fetch *before* sleeping 10s (`HAController.swift:91–94`), so a stalled fetch will delay the next poll. Acceptable for read-only.

What it costs:
- Battery on the iPad is irrelevant — wall panels are perpetually on AC.
- Perceived UI freshness on user-driven changes (you flip a switch in the HA app, the wall panel may be 0–10s behind).
- HA load: one `/api/states` GET every 10s per panel × 2 panels × 339 entities is trivial.

When does WebSocket migration become urgent? **The moment writes ship.** With writes, the user taps a light tile → optimistic update → real HA write → server pushes the actual final state back. With polling, the panel will see its own optimistic value for up to 10s before the polling loop confirms or contradicts it. Race conditions between concurrent panels (Mei changes a light from her phone while Todd taps the same tile on the wall) become user-visible flicker. WebSocket isn't a Phase 0d.3 polish — it's Phase 0d.2 + 1 week.

Worth noting: `stateChanges` is already on the protocol as an `AsyncStream<StateChangeEvent>` (line 68) but there is **zero consumer code** anywhere in the app. `PollingHAClient` returns an immediately-finishing stream (line 92). When 0d.3 implements the WebSocket, `HAController` needs to actually subscribe to it. The subscription pattern is unwritten.

### A.4 — Layer 1 / Layer 2 framing in code

**Verdict: Sound. Stubs are in the right places and typed correctly.**

`HAController.swift:165–205` declares eight write stubs:
```
setThermostatMode, setThermostatTarget, setEcoPreset,
setRoomBrightness, toggleRoom, setLightChild,
toggleFrontDoorLock, toggleGarage, setHomeMode
```

All throw `HAError.notImplemented`. Call sites are pre-wired: `MainPanelView.swift:153/172`, `BottomActionRow.swift:127/150`, `LightDrillInView.swift:119`, `StatusBarView.swift:237` — every interactive control already routes through `model.notifyWriteDisabled()` which fires the toast.

When 0d.2 begins, the work is replacing each stub body with:
1. Validate inputs (range checks for brightness 0..1, temperature range)
2. Apply optimistic update to `home` inside `withAnimation`
3. `try await haClient.callService(domain:, service:, data:)`
4. On throw: silently log; next poll corrects state

Step 3 is the typed-payload concern from A.2.

The `notifyWriteDisabled` toast is currently the entire Layer 1→Layer 2 transition surface. Once writes are real, that helper deletes; views call the controller method directly. Refactoring upstream is unnecessary because the call-site shape is already a method invocation, not a conditional.

### A.5 — VoiceAssistant protocol abstraction

**Verdict: Wrong shape — half of it is dead.**

The protocol defines two methods (`VoiceAssistant.swift:11–24`):
- `respond(to:context:) async throws -> VoiceAssistantResponse`
- `streamResponse(to:context:) -> AsyncThrowingStream<String, Error>`

Grep for callers of `respond(`: only the protocol declaration and the implementation. **Nothing in the app calls the non-streaming variant.** `VoiceCoordinator.runPipeline` calls `assistant.streamResponse` on line 155.

The protocol's purpose is "future LLM swaps are trivial." A future SambaNova implementation would have to provide both — which means writing a non-streaming wrapper that nobody uses, or violating the protocol. Either drop `respond()` entirely (the streaming variant subsumes it) or keep it and document it as the bundled-phrase fallback path (which doesn't exist as written — see A.7). Right now it's documented as "kept for compatibility" in `ClaudeVoiceAssistant.swift:12` — compatibility with what isn't specified.

The streaming variant is well-shaped: returns an `AsyncThrowingStream<String, Error>` of token deltas, which is what Anthropic SSE produces and what `CartesiaWSClient.feedToken` consumes. Future swap to SambaNova requires producing the same stream from a different SSE endpoint — straightforward.

### A.6 — Streaming pipeline (LLM SSE → Cartesia WS → AVAudioEngine)

**Verdict: Sound but with caveats. Right architecture, wrong coordination.**

Three-layer streaming is the correct architectural choice. The latency math is real: full-response synchronous (LLM done → TTS done → audio start) is ~1100ms at Haiku 4.5; chunked streaming target is ~750ms. The brief's pipeline rationale (Section 11.5–11.6) is internally consistent.

What's wrong is the coordination, not the layers. `VoiceCoordinator.runPipeline` (lines 102–268) interleaves five concerns into one ~170-line function:
1. STT lifecycle
2. Tick playback (Stage 2)
3. LLM streaming + Cartesia feeding (lines 171–206 — nested Tasks-in-Tasks)
4. Tick wait gate (Stage 4 — the gate CC and AUC both flagged)
5. Audio playback (Stage 5)

The control flow isn't quite a graph and isn't quite linear. Lines 191–205 nest a `Task { @MainActor [weak self] in ... }` inside another `Task<Void, Error> { [weak self] in ... }`, with both ending up assigned to `audioStreamTask` and `llmTask` in mismatched scopes. This works, but reasoning about cancellation, error propagation, and the `Task.isCancelled` check at line 219 across two nested Tasks is harder than it needs to be.

Simpler alternative that preserves the streaming win: extract the Cartesia-feeding loop and the audio-buffering loop into typed helpers (e.g. `CartesiaPipe.feed(_ stream: AsyncThrowingStream<String>) async throws -> AsyncStream<Data>`), and let `runPipeline` await sequentially:
```
let textStream = assistant.streamResponse(...)
let audioStream = try await cartesia.feed(textStream)
for await chunk in audioStream { audioPlayer.schedule(chunk) }
```

This is the same architecture; the difference is that the parallelism is encapsulated inside the helpers, not visible at the orchestration layer.

What synchronous costs would be: ~350ms more perceived latency; what it'd buy: a dramatically simpler pipeline. I would not regress to synchronous, but the current implementation's complexity is paying for ~350ms of latency improvement, which is right at the edge of "is the complexity worth it." After lived testing, if the perceived improvement is solid, keep it. If it's marginal, consider simplifying.

### A.7 — Bundled phrases vs runtime TTS

**Verdict: Sound but with caveats. The split is right; the dead code around it is not.**

The split: bundled `.m4a` files (rendered at build time via `tools/render-phrases.py`) for canon/transition/tick pools (199 phrases); runtime Cartesia TTS for the conversational responses. This is the right shape — pre-rendered audio for fixed content, runtime TTS for dynamic. It minimizes Cartesia spend on repeat phrases and gives offline resilience for transitions.

Where it breaks:
- `CartesiaTTSClient.swift` (HTTP non-streaming) is in the build but not called from anywhere in app code. Comments in `CartesiaWSClient.swift:8` and `AudioStreamPlayer.swift:18` say it's "kept for bundled phrase fallback" — but bundled phrases are pre-rendered shipping as `.m4a` files; they don't go through Cartesia at runtime. CartesiaTTSClient is dead.
- `VoiceService.playRawAudio` (line 346) accepts raw MP3/m4a `Data` via `AVAudioPlayer(data:)` — this was the pre-streaming path used in 0c.17. Streaming path uses `AudioStreamPlayer` instead. `playRawAudio` has zero callers.
- `VoiceService.speakRandomPhrase` (line 335) is documented as a "legacy alias" wrapping `handleEvent(.micTap)`. It's exposed through `AppModel.speakRandomPhrase()` (line 141). Grep finds zero view callers. Dead surface.

When does the bundled approach hit a wall? When phrase-like state-change responses ("Lights dimmed in the kitchen, sir") become dynamic enough that pre-rendering combinations is impractical. The brief's Phase 1.7 (runtime TTS expansion) is exactly that boundary. The current Cartesia WS pipeline already handles dynamic synthesis; 1.7 is wiring it to non-conversational triggers.

### A.8 — Two parallel audio playback paths

**Verdict: Wrong or drifting. The two-paths claim is true; the rationale is not.**

Today: `AVAudioPlayer` (in `VoiceService` for bundled phrases) + `AVAudioEngine` + `AVAudioPlayerNode` (in `AudioStreamPlayer` for streaming). The comment in `AudioStreamPlayer.swift:18–19` says: *"The non-streaming VoiceService.playRawAudio() (AVAudioPlayer / mp3) is kept for bundled phrase pool — those files are m4a, which AVAudioEngine can't decode inline."*

That justification is fine for **bundled .m4a phrases** played via `makePlayer` (`VoiceService.swift:398`), which is real and active code. It's not the path `playRawAudio` is on — `playRawAudio` is dead.

So the two paths are:
- **Bundled .m4a path:** `AVAudioPlayer(contentsOf:)` from bundle URL — used for transition pools, mic-event pool, thinking ticks. Active.
- **Streaming PCM path:** `AVAudioEngine` + `AVAudioPlayerNode` with manually scheduled buffers — used for Cartesia WS responses. Active.

These can stay separate. They don't share state, they don't share audio session ownership cleanly (each grabs `.playback` independently — `AudioStreamPlayer.swift:92–94`, `VoiceService.swift:181–182`), and they don't share lifecycle. Audio session contention between them is what caused the wake-word coordination work in 0c.19.

The right move is not unification — the right move is making session ownership explicit. Today, three classes (`AudioStreamPlayer`, `VoiceService`, `AppleSpeechCoordinator`) all call `AVAudioSession.sharedInstance().setCategory(...)` independently. Add a small `AudioSessionCoordinator` that all three go through. Pre-0d.2 polish, post-verification.

### A.9 — On-device STT (Apple SFSpeechRecognizer) vs cloud STT

**Verdict: Sound.**

`AppleSpeechCoordinator` requires on-device recognition (`requiresOnDeviceRecognition = true`, line 79). Free, private, no network dependency, latency 200–800ms. The known weakness is accuracy on unusual proper nouns ("Baxter" occasionally transcribed as "Baxter," "Backster," "Boxter") but this is irrelevant for the conversational layer because the LLM will normalize. For wake-word the answer is Porcupine (separate model), so SFSpeechRecognizer's accuracy ceiling doesn't constrain.

Alternative would be Deepgram or Whisper API — better accuracy, ~150ms more latency, network-dependent, $0.005-ish per turn. Not worth the trade for a household-scale tool.

### A.10 — Wake word as separate subsystem (Porcupine)

**Verdict: Sound. Coordination cost is real but contained.**

Porcupine is the right call for "always-listening, low-power, on-device wake word." SFSpeechRecognizer has an always-listening mode but with much higher CPU cost and a battery story that doesn't matter on an AC-powered iPad — except it DOES create thermal load on a wall-mounted device that's also rendering animations.

The coordination cost is what `WakeWordDetector.swift` is paying. State machine (`stopped`/`running`/`paused`), pause-at-pipeline-entry, resume-at-pipeline-exit, audio session handoff via full engine teardown (`stopEngine()` line 214). The implementation is correct. The pause/resume contract is documented and consistent.

The architectural risk worth flagging: `WakeWordDetector.pause` is called at five distinct points across the code (`VoiceCoordinator.runPipeline:108`, `VoiceService.handleEvent` for both mic and panel paths, `VoiceService.playRawAudio:348` if it were ever called, `VoiceService.cancelModal:330`). Resume is similarly scattered. Any future code path that plays audio without going through these helpers will silently leave wake word paused. This is the AudioSessionCoordinator concern from A.8, generalized.

### A.11 — System prompt as bundled markdown file with versioning

**Verdict: Sound.**

`baxter-system-prompt-v2.md` in `WallPanel/WallPanel/Resources/`, loaded by `ClaudeVoiceAssistant.loadSystemPrompt` (line 93). v1 is also in the bundle as historical record. Sync script `tools/sync-character.sh` mirrors to public repo.

Comparing alternatives:
- **Swift constants:** rebuild on every iteration; no diff-friendly history; harder for non-coder iteration. Worse.
- **JSON config:** structured but loses readability. Markdown is the right text format for a prompt.
- **Remote fetch:** introduces a network dependency on a piece of state critical to the assistant's character. Bad idea.

The bundled-markdown approach is correct. The hardcoded resource name `"baxter-system-prompt-v2"` (line 95) is the only smell — every prompt version bump requires a code edit. Trivial to fix later via `VoiceAPIConfig.systemPromptResource` constant.

The hardcoded fallback prompt (lines 102–106) at `ClaudeVoiceAssistant.swift` is what shipped in builds 67–70 and is the `"You are Woadhouse"` capsule that AUC and CC discussed. Mark this for deletion after build 71 verification — the file-not-found path should now be a hard `fatalError()` in DEBUG, not a silent fallback. Silent fallback is what hid the "v1 never in bundle" bug for four builds.

### A.12 — Keychain for API key storage with seed-and-remove

**Verdict: Sound.**

`VoiceAPIConfig` (lines 39–59) reads three keys from Keychain via `KeychainHelper.load(key:)`. Same pattern for HA token. The seed-and-remove dance is awkward but secure: keys never enter the source repo, they survive reinstalls, and you can revoke the device's access to the keys without touching the app. For a personal-use wall panel this is appropriate.

Two real-world wrinkles worth knowing:
- After a Keychain reset (rare but possible — device restore from old backup, iOS Keychain bug), the app silently logs "API key not configured" and runs in degraded mode. There's no UI surface to detect or re-seed. Settings UI for key state ("Anthropic: configured ✓ / Cartesia: not configured ✗ — re-seed via Xcode") would resolve.
- The `picovoiceConfigured` check (line 59) in `AppModel.startWakeWord()` (line 97) gates wake word, but doesn't gate the build. If wake word is later required for the panel to function, the silent-noop pattern hides the misconfiguration.

---

## Section B — Subsystems not covered by prior audits

### B.1 — HAClient implementation walk

`PollingHAClient.fetchAllStates` (lines 97–119) does one HTTP GET per poll; on 401 throws `unauthorized`; otherwise parses the entire JSON array. No retry logic. No backoff. If HA is unreachable, every 10s the user gets a fresh failure, the connection state flips to `.error(...)`, and on next success it flips back to `.connected`. This works but produces UI flicker — the connection indicator will toggle whenever the network blips for a few seconds.

Malformed responses: `JSONSerialization.jsonObject(with: data) as? [[String: Any]] ?? []` (line 111) silently returns an empty array on any structural failure. The panel will then show the entire home as "fall back to hardcoded data" via `HomeState.makeHardcoded()` (line 125, 176) — *the panel will silently show fake state*. This is a real problem for 0d.2: imagine a malformed HA response leading the panel to display a hardcoded 68° target, the user taps "set to 70°," the write succeeds against a thermostat that was actually at 65°. The fallback should be `lastKnownGood`, not `hardcoded`.

`entityHealth` reporting (line 72) is good — it counts resolved-vs-configured entities. But this is a sum, not a breakdown. The Settings UI shows "8/9 resolved" but not "garage entity missing." When 0d.2 ships, knowing *which* entity is unresolved becomes operationally important.

What happens when HA is unreachable for an extended period: poll loop continues; `connect()` is only called once at start; if the network was up at startup but went down later, `isConnected = false` is never set (line 56's `disconnect()` is only called on `stopPolling`). The connection-state UI is driven by `haConnectionState` in `HAController`, not `haClient.isConnected`. They can disagree.

### B.2 — SwiftUI view hierarchy

Top-level: `RootView` (122 lines) → `AmbientView` + `MainPanelView` + `TranscriptBand` + write toast + `VoiceModalOverlay` overlaid in a ZStack. `MainPanelView` (~250 lines) decomposes by panel state (`QuickControlContent` and `FullControlContent` as private structs). `FullControl` further decomposes into `ClimateSection`, `RoomsSection`, `BottomActionRow` (each in its own file).

This decomposition is appropriate. View files are 100–250 lines, each focused on one concern. The file split mentioned in 0c.10 ("Split MainPanelView.swift into 6 logical files") delivered.

Leaks I found:
- `RootView.swift:90`: `model.startWakeWord()` is called from `.onAppear`. So is `startPhotos`, `startWeather`, `startHA`. These four startup calls should be one `model.start()` method — the view shouldn't be the startup orchestrator.
- `MainPanelView.swift:221`: `UIScreen.main.bounds.width` used inside a `GeometryReader`-enclosed VStack. The geometry proxy is in scope; use it.
- `RootView.swift:11–16`: `showVoiceModal` is computed from `voicePresentationState` — a view-layer derivation of business logic. Could live in `VoiceService` as a computed property.

### B.3 — Hearth design system

`HomeControlCore/Design/HearthColors.swift`, `HearthSpacing.swift`, `HearthTypography.swift`, plus runtime-sized tokens in `HearthLayout` and `HearthAnimation` enums. All values are Swift constants/enums, no JSON. Color extensions hang off `Color` itself; spacing/touch-target/animation in dedicated enums.

The discipline is internally consistent. Hex initializer is a tidy pattern. The `tileFill`/`tileBorder`/`pillFill`/`pillBorder` opacity helpers (`HearthColors.swift:68–77`) consolidate exactly the kind of "0.07 / 0.15 / 0.25 scattered across 5 files" smell that erodes design systems over time.

Path for new tokens is clear: add to the appropriate enum, update brief Section 5 if it's structural. No hardcoded magic values found in the views I read except the layout glitch at MainPanelView:221.

### B.4 — Photo subsystem

`PhotoController.swift` + `HomeControlCore/Photos/PhotoLibrary.swift` + `ShuffleQueue.swift`. Reads from local Photos via `PHPhotoLibrary` — no network. iCloud Shared Album integration happens at the Photos-framework level: when the album is iCloud-shared, iOS handles sync transparently, but `loadImage` with `isNetworkAccessAllowed = true` (line 126) means images may be fetched on demand from iCloud.

What happens when network is intermittent: `PHImageManager.requestImage` returns nil if the asset isn't downloaded and network is unavailable. `advancePhoto` (line 109) sets `currentImage = img` — which sets it to nil. The view shows nothing on screen. **Silent failure.** No retry, no error state, no indication. Same silent-grace pattern as voice.

Rotation loop self-gates on `panelState == .ambient` (line 132) — correct. But the loop sleeps 5 minutes regardless; if the user is engaged for 10 minutes then dismisses, they wait up to 5 more minutes for the next photo. Minor.

Slight rotation in `TranscriptBand` (the ±0.5° "handwritten note" feel): not in `PhotoLibrary`, in `TranscriptBand.swift`. Worth verifying once on device — DC's audit flagged "may read as glitch" and that lived call hasn't been made.

### B.5 — Phrase render pipeline

`tools/render-phrases.py` reads `tools/phrases.json`, calls Cartesia HTTP, writes `WallPanel/WallPanel/Audio/phrase_NNN.m4a`, hash-caches in `.phrase_hashes.json`, auto-copies the manifest to `WallPanel/WallPanel/phrases.json`. The hash cache is correct: only re-renders when the text or pool changes. `--force` re-renders all.

Build integration: **manual.** The script is not invoked from Xcode build phases. If someone changes a phrase in the manifest and forgets to run the script, the build succeeds with stale audio (silently). If someone adds a new phrase number to the manifest without running the script, the file isn't in the bundle and `pickPhrase` will skip it via the `poolURLs` precomputation.

Two improvements possible: a Run Script Build Phase that runs `render-phrases.py --dry-run` and fails if anything would re-render, and a CI check that runs a non-dry render on PRs to the manifest. Neither is essential for a personal project, but the dry-run hook is a 10-minute change and prevents the failure mode.

### B.6 — Logging and telemetry

The codebase uses bare `print()` statements throughout. `[Baxter]` prefix unification per CC's audit is partial — `CartesiaWSClient.swift` and `AudioStreamPlayer.swift` still use `──` prefix (the commit `5a9d012 chore: unify telemetry prefix` apparently missed them).

There is no logging abstraction — no `os.Logger`, no `OSLog`, no levels. This means:
- No structured filtering beyond grep on Console.app
- No automatic redaction of sensitive values
- No log persistence across launches
- All logs ship in release builds (no `#if DEBUG` gates on most prints)

For a personal-use wall panel, `print()` is defensible. For a project that's already accumulated `[Baxter]`, `[WeatherService]`, `──`, and various tag prefixes, it's becoming inconsistent enough to warrant a real `Logger`. `import os` and a `Logger(subsystem: "com.tramel.homecontrol", category: "voice")` would unify, give automatic redaction, and let Console.app filter by category natively.

`VoiceInteractionTelemetry` (timestamp dictionary T1–T10) is the right shape for performance telemetry and the `#if DEBUG` gate (`VoiceCoordinator.swift:260`) appropriately gates its costly logging. Good.

Things that should log but don't:
- Empty transcript drop (CC flagged)
- Audio interruption events (CC flagged)
- WakeWordDetector silent death after failed resume (CC flagged)
- HA polling failure → fallback-to-hardcoded path (B.1 finding above)
- Photo asset load failures (B.4)

### B.7 — PanelConfig system

Compile-time configs declared as `static let firstFloor`, `secondFloor`, `personal` in `PanelConfig.swift`. Selection via `PanelConfig.current = firstFloor` at line 194 with a comment showing the future `#if PANEL_SECOND_FLOOR ... #elseif PANEL_PERSONAL` pattern. Today **all builds resolve to firstFloor.**

Per the brief, this is by design — separate Xcode schemes will set different `current`. The pattern scales: adding a kitchen panel is a new `static let kitchen = PanelConfig(...)` and a new scheme. The `Sendable` conformance on `PanelConfig` and its enums means it crosses actor boundaries cleanly (used in `PollingHAClient.init`).

What I'd change: the `#if PANEL_X` selection mechanism is comment-only right now. The 2nd-floor scheme work should land *with* the scheme, not in advance. This is a minor "code says one thing, build says another" issue — currently `current` always returns `firstFloor` regardless of scheme.

### B.8 — Build, signing, deployment

`deploy.sh` is a 40-line shell script: `xcodebuild -workspace -scheme WallPanel -configuration Debug -destination platform=iOS,id=$IPAD_PRO`, then `xcrun devicectl device install app --device $UDID`. UDIDs hardcoded for iPad Pro and iPad Air.

I ran `xcrun devicectl list devices` during this audit. Result:

```
Todd LaVigne's iPad   ...  8B58895C-4976-...  unavailable     iPad Pro 12.9"
Todd's iPad Air       ...  25423385-294F-...  unavailable     iPad Air M2
Todd Daytime Watch 2  ...  A324A285-...       available (paired)
Todd LaVigne's iPhone ...  CD6769D0-...       available (paired)
```

The iPhone and Apple Watch show "available (paired)" — confirming wireless device deployment over the network *does* work via `devicectl` once a device is paired. AUC's claim that wireless deploy is supported is correct.

**However**, both iPads show "unavailable." That doesn't mean wireless isn't supported — it means the iPads are not currently reachable on the LAN (probably asleep, not on the WiFi network the Mac sees, or in a state where the trust isn't being honored). The fix is operational, not architectural: wake the iPad, ensure it's on the same LAN, optionally force re-trust by connecting via USB once. Once paired and reachable, `devicectl device install app --device <UDID> <path>` works without USB.

So AUC's "10-minute fix" is too optimistic if the iPads' pairing state has degraded. But the architectural claim is right: this project does not require physical USB connection for every install.

The deploy.sh script doesn't surface the unavailable state — it'll just fail with a confusing error. A pre-flight check (`devicectl list devices --filter "Name CONTAINS 'iPad'"` and assert state == "available") would make failures actionable.

### B.9 — Tests

Zero test infrastructure. No `Tests/` directory in `HomeControlCore`. No `WallPanelTests` target. No XCTest files. The handoff doc and completion reports never mention adding tests; the only "test" pattern is lived use on device.

Highest-value tests if any were written:
1. `PollingHAClient` parsers (`parseClimate`, `parseRooms`, `parseHouseStatus`, `parseHomeMode`) — input is a `[String: Any]` dict, output is a Sendable struct. Trivial to feed a fixture JSON and snapshot the output.
2. `ShuffleQueue` — pure data structure, no system dependencies.
3. `CartesiaWSClient.hasSentenceBoundary` — pure function over text.
4. `VoiceCoordinator.appendHistory` (the 6-turn sliding window) — pure logic.

What's hard to test as currently architected:
- `runPipeline` is one ~170-line `MainActor` async function tightly coupled to `AppleSpeechCoordinator`, `VoiceService`, `CartesiaWSClient`, `AudioStreamPlayer`, `WakeWordDetector`. There's no seam for injecting fakes. The `VoiceAssistant` protocol *is* a seam for the LLM; nothing else has one.
- `PollingHAClient` calls `URLSession.shared` directly. No `URLSessionProtocol` abstraction; can't intercept in tests.

The architecture's testability is below what I'd expect for code that has a `VoiceAssistant` protocol. Adding `HTTPClient` and `WSClient` protocols would mirror the same pattern and make `PollingHAClient` and `CartesiaWSClient` testable without rewriting them.

For a personal-use wall panel with one human reviewer, "we test in lived use" is defensible. For a project with three Claudes, four external dependencies, and an emerging Layer 2 control surface, having zero tests means every regression is caught (or not) by Todd. That's a steering signal as thin as the one AUC flagged.

---

## Section C — Architectural debt map

### C.1 — Where the brief and the code disagree

- **Wake word.** Brief Section 17 lists it as non-goal #1. Phase 0c.19 (`WakeWordDetector.swift`) implements it. AUC flagged this; recording it here for the architecture map.
- **Conversation history persistence.** Brief Section 11.9: "Conversation history — last 6 turns ... Resets on app launch. No persistence, no long-term memory." Code matches. But Layer 2 + tool-use integration in Phase 2 will inevitably need cross-session memory (otherwise "remind me about the dishwasher" becomes useless). The brief's "no long-term memory" is a Layer 1 commitment that dissolves at Layer 2. Worth flagging as a future scope decision, not a current drift.
- **System prompt name and identity.** v2 still says "You are Woadhouse." Brief uses "Baxter." AUC and CC both flagged.
- **`speakRandomPhrase`, `playRawAudio`, `CartesiaTTSClient`, `respond()`.** All four are in the build but called by nothing. Brief implies a layered architecture; reality is that 30% of the voice surface area is dead code.
- **Hereto-be-determined.** Brief 11.5 says "Cartesia output_format ... mp3, 44100Hz" in `VoiceAPIConfig.swift:25` — but the actual streaming WS uses `pcm_s16le` at 44100Hz. Comment is stale.
- **Brief Section 5.4:** "Status bar must be compact. ~100pt tall." `HearthSpacing.swift:28`: `static let height: CGFloat = 130`. Code drifted to 130; brief still says 100. Pick one.
- **Brief Section 7:** "Quick control must stay at ~22% vertical max." `HearthSpacing.swift:40`: `maxVerticalFraction: CGFloat = 0.40`. Drifted to 40%. Same issue.

### C.2 — Debts that 0d.2 will compound

1. **`[String: any Sendable]` payload typing in `callService`** — see A.2. Replace with typed `Encodable` wrapper before first write ships.
2. **Polling-only state freshness** — see A.3. WebSocket should land *with* writes, not after.
3. **Fallback-to-hardcoded data on parse failure** — see B.1. Replace with `lastKnownGood` snapshot, or write will dispatch against fictional state.
4. **No optimistic-update timeout** — `applySnapshot` (HAController:122) replaces home state on every poll. After a write, between optimistic update and next poll, if the write succeeds *and* HA reports back fast (sub-10s), there's no race. If HA is slow, the optimistic value sticks; if HA reports a different value, the optimistic update is silently overwritten. There's no "if the value is still the optimistic one after 5s, mark it as failed" path. This is exactly the kind of state-sync issue that becomes user-visible when writes can fail.
5. **No write-feedback discipline** — every error path in `runPipeline` is "silent grace." When 0d.2 wires writes, "tap a light, nothing happens" with no transcript and no toast is worse than current "tap a light, write toast says coming soon." Need explicit success and failure feedback.
6. **Concurrent pipeline guard incomplete** — `VoiceCoordinator.startConversation` line 76 guards on `isListening`, which is `true` only during STT. After STT finalizes (line 126) and before audio playback completes, a second wake-word fire WILL trigger a second pipeline. Wake word is currently inactive so this is latent. With wake word + writes, "Baxter, lights on. Baxter, lights off. Baxter, lights on." said quickly produces a race on the audio session that today is masked.

### C.3 — Dead architecture

Listed by impact:

1. **`CartesiaTTSClient.swift`** — entire file (86 lines) has zero non-self-referential callers.
2. **`VoiceService.playRawAudio`** (line 346) — zero callers.
3. **`VoiceService.speakRandomPhrase`** (line 335) and the corresponding `AppModel.speakRandomPhrase` forward — legacy alias. The `.micTap` event branch in `handleEvent` (lines 222–266) reads pools `.canonMain` and `.casualMic`, which are 50/50 weighted but only fired through the dead alias. The branch is dead too.
4. **`VoiceAssistant.respond()`** — zero callers; `streamResponse()` is the only used method.
5. **`HAClient.stateChanges`** — zero subscribers; protocol surface waiting on 0d.3.
6. **`MockHAClient`** — used only as the initial value in `HAController` before `startPolling` swaps it. It's not actually a fallback in failure cases (HA failure flips connection state, doesn't swap clients). Used to be the test path; in a project with no tests, its job is purely the "haven't connected yet" placeholder.
7. **Stale comment block in `VoiceAPIConfig.swift:25`** — claims output_format is mp3, but conversational pipeline is pcm_s16le.
8. **`woadhouse-system-prompt-v1.md`** still in bundle target despite the rename.

Cleanup deletes ~150 lines of dead Swift, simplifies the voice subsystem's mental model substantially, and removes four "wait, what's this for?" speed bumps for any future reader.

---

## Section D — Cross-cutting concerns

### D.1 — Concurrency

`HomeControlCore` package: `-strict-concurrency=complete` enabled (`Package.swift:27`). Clean.

`WallPanel` app target: **not enabled.** `pbxproj` does not contain `SWIFT_STRICT_CONCURRENCY=complete`. The handoff doc claims "Strict concurrency complete + zero warnings" — that's true for the core package only. Most of the voice subsystem and all the views are in the app target, so they compile under the looser default checking. This is fine for now (the actor/MainActor annotations are deliberate) but means the "zero warnings" bar is lower than it sounds. Promoting the app target to strict-complete is a real day of work; defer until the dead code is cleaned out.

Sendable boundaries: clean. `HomeSnapshot`, `EntityHealth`, `StateChangeEvent`, `PanelConfig`, `VoiceTurn` all explicitly Sendable. `[String: any Sendable]` is the only weak point (A.2).

MainActor crossings: explicit and consistent. Every async await crossing the MainActor boundary uses `Task { @MainActor in ... }` or implicit isolation. No found instances of `assumeIsolated` or other escape hatches.

### D.2 — Error handling philosophy

Consistent in shape: `LocalizedError` enums per subsystem (`HAError`, `WSError`, `APIError`, `TTSError`, `WakeWordDetectorError`). `errorDescription` provided.

Inconsistent in handling: voice pipeline errors are silent-grace; HA errors go to `haConnectionState`; weather errors set `weatherError` and `lastError`; photo errors are silently nil; render-pipeline errors print and continue. Five different patterns.

The right discipline is a small `AppError` envelope that carries: subsystem, severity (silent / log / user-visible), recovery suggestion. Today's "every layer makes its own choice" pattern is workable for a single-author personal project but already producing surprises (CC noted the silent-WS-disconnect; B.1 noted the silent-HA-malformed; B.4 noted the silent-photo-load-failure).

### D.3 — Memory and lifecycle

Object lifetimes are mostly correct. Weak-reference patterns appropriate:
- `WakeWordDetector` weak refs in `VoiceCoordinator` (line 52) and `VoiceService` (line 167) — correct, AppModel owns it.
- `[weak self]` consistently used in escaping closures — `AppleSpeechCoordinator.swift:88, 95, 112, 143`; `WakeWordDetector.swift:200–204`; `AudioStreamPlayer.swift:135–137`; etc.
- `PhotoLibrary.deinit` correctly unregisters the observer (line 50).

Lifetimes I'd flag:
- `VoiceCoordinator.activeTask` (line 62) is replaced on every `startConversation`. The previous task is cancelled but `activeTask` is then reassigned — if `cancel()` doesn't immediately stop the previous task (it doesn't; it only sets the cancellation flag), and the previous task hasn't yet hit a cancellation check, two tasks briefly coexist. Brief overlap is harmless if both honor `Task.isCancelled`, but the audio session ownership during overlap is undefined.
- `CartesiaWSClient` is created fresh per turn (line 157) and `close()` is called in some but not all exit paths of `runPipeline`. The `defer { close() }` pattern would be safer than the current "cancel and forget."

### D.4 — Threading model

Audio threads, UI thread, polling. `VoiceCoordinator`, `VoiceService`, `AppleSpeechCoordinator`, `AudioStreamPlayer`, `WakeWordDetector`, `ClaudeVoiceAssistant`, `HAController`, `PhotoController`, `PanelStateController`, `AppModel`, `WeatherService` — **everything** is `@MainActor`. The only off-main work happens inside `URLSession` and AVFoundation callbacks, which marshal back to main via `Task { @MainActor in ... }`.

This is a deliberate choice and it's the right one for SwiftUI + Observation. Performance cost is negligible at human-input rates. The model is coherent.

The exception is `HAClient` (and `PollingHAClient`), which is its own actor. Cross-actor `await` calls are explicit and constrained. Coherent.

The audio engine threads are below the visibility horizon — AVFoundation owns them. The tap callback on `WakeWordDetector.swift:200` runs on the audio hardware thread; `convertToInt16` is `static` and operates on value types (line 225–261); `processSamples` is dispatched to main via `DispatchQueue.main.async`. Correct.

### D.5 — Resource management

What's acquired and released:
- **Audio sessions:** acquired by three classes independently (A.8). Released implicitly via category changes. No central coordinator.
- **Network connections:** `URLSession.shared` reused across HTTP calls (`PollingHAClient`, `WeatherService`, `AnthropicAPIClient`, `CartesiaTTSClient` if it were called). WebSocket per turn in `CartesiaWSClient` — CC's finding.
- **File handles:** Bundle URL lookups + `AVAudioPlayer(contentsOf:)`. AVFoundation manages handle lifecycle; nothing leaks here.
- **Tasks:** `pollingTask`, `weatherTask`, `toastTask`, `quickControlTask`, `fullControlTask`, `maxDurationTask`, `pipelineTask`, `audioStreamTask`. All `private var`, all canceled before reassignment. Mostly clean. `runPipeline`'s nested Task dance (A.6) is the one place where ownership-of-ownership is muddled.
- **Continuations:** `CheckedContinuation` patterns in `AppleSpeechCoordinator.swift:35, 162` and `AudioStreamPlayer.swift:42`. CC flagged hang risks; resource-management impact is the same — a never-resumed continuation is a permanent suspension.

Wastefully reacquired: WebSocket per turn (CC, fixable). Audio engine in `AudioStreamPlayer.start()` created fresh every turn and stopped at completion (lines 83, 125). For sub-10-turn voice sessions this is fine; for an always-on conversational panel handling dozens of turns, the `AVAudioEngine.start/stop` cycle is non-trivial work. Acceptable today; revisit if 0d.3 enables more conversational density.

---

## Top 10 findings by impact

1. **Replace `[String: any Sendable]` with a typed `Encodable` payload before 0d.2 ships writes.** This is the biggest pre-Layer-2 architectural decision. **Priority: Critical | Effort: 4 hours | Risk: Medium**

2. **WebSocket migration is Phase 0d.2 + 1 week, not Phase 0d.3.** Polling-only writes will produce optimistic-update flicker and concurrent-panel races. Promote 0d.3 ahead of 1.6/1.7/1.8. **Priority: High | Effort: 1–2 days | Risk: Medium**

3. **Replace HardcodedData fallback in `PollingHAClient` with `lastKnownGood`.** Today, malformed HA responses produce a panel showing fictional state — the panel disagrees with the home. **Priority: High | Effort: 2 hours | Risk: Low**

4. **Delete dead voice code.** `CartesiaTTSClient`, `VoiceService.playRawAudio`, `VoiceService.speakRandomPhrase`, `VoiceAssistant.respond()`, the `.canonMain`/`.casualMic` micTap branch, woadhouse-system-prompt-v1.md from bundle. ~150 lines, simplifies mental model. **Priority: High | Effort: 2 hours | Risk: Low**

5. **Add `AudioSessionCoordinator`.** Three classes claim `.playback`/`.record` independently; coordination is implicit through ad-hoc `pause`/`tryResume` scattered across five call sites. **Priority: Medium | Effort: 4 hours | Risk: Medium**

6. **Promote app target to `-strict-concurrency=complete`.** Core package is clean; app is the larger surface. Cleaning the dead code first reduces this from "day of work" to "afternoon." **Priority: Medium | Effort: 4–8 hours | Risk: Low**

7. **Convert all `print()` to `os.Logger` with subsystem/category.** Native Console.app filtering, automatic redaction, retention. **Priority: Medium | Effort: 3 hours | Risk: None**

8. **Extract `runPipeline` into named stage helpers.** The function is ~170 lines with two nested Tasks; the streaming win is real but the orchestration complexity is paying for ~350ms latency that the user can't see during the audit. **Priority: Medium | Effort: 4 hours | Risk: Low**

9. **Make `loadSystemPrompt`'s file-not-found path a `fatalError` in DEBUG.** Silent fallback hid the v1-not-bundled bug for four builds. Don't let it happen again. **Priority: Low | Effort: 5 minutes | Risk: None**

10. **Add a pre-flight check to `deploy.sh`.** Assert `devicectl list devices` shows iPad as "available" before attempting install — fail fast with an actionable message. **Priority: Low | Effort: 30 min | Risk: None**

---

## Five things to NOT change

1. **The `HAClient` actor + protocol shape.** Will carry 0d.2 and 0d.3 cleanly. The Sendable disciplines, the `HomeSnapshot` value type, the `connect/fetchHome/callService/disconnect` lifecycle are all correctly drawn.

2. **The `@MainActor` everywhere convention.** The cost is negligible; the reasoning simplification is significant. Don't be tempted to "move things off main for performance" without measured evidence.

3. **The compile-time `PanelConfig` pattern.** Resists the gravitational pull toward in-app settings screens. Adding new panels is a new `static let` and a scheme.

4. **The Hearth design system in Swift constants.** No JSON tokens, no hot-reload config — just typed `Color` extensions and `enum` namespaces. Boring is right here.

5. **Bundle-shipped system prompt + sync script + character versioning.** This is the one piece of the project where the artifact-heavy workflow is genuinely load-bearing. The prompt is the soul of the assistant; versioning it like code is correct.

---

— TAuC, end of technical architecture audit
