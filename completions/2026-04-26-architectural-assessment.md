# Architectural Assessment — Home Control Panel

**Written:** 2026-04-26, end of Phase 0c.10  
**Context:** v0.4.0, post-foundation-refactor, pre-writes. Read-only HA over REST polling. No voice, no alerts, no WebSocket.  
**Framing:** We're closer to the start than the end. What decisions need to happen before we keep stacking features?

---

## 1. The HAClient Protocol — Is It Right for What's Coming?

### Current shape

```swift
protocol HAClient {
    func fetchHome() async throws -> HomeSnapshot
}
```

Two implementations: `MockHAClient` (hardcoded) and `PollingHAClient` (REST). `PollingHAClient` is a `final class`. `HAController` owns a polling Task loop.

### When WebSocket arrives (Phase 0d.3)

The current protocol **does not fit WebSocket**. WebSocket is an event stream, not a request-response. The `fetchHome()` call maps naturally to REST polling — call it, get all state, done. WebSocket would push per-entity state change events, not full-home snapshots.

**The mismatch:** With REST polling, the client is passive — the controller asks, the client answers. With WebSocket, the client is active — the client pushes events, the controller listens. A protocol shaped for REST polling forces the WebSocket implementation to either (a) buffer events and batch them into fake `HomeSnapshot` responses on a timer, or (b) require a protocol redesign before WebSocket can ship.

**What needs to happen before 0d.3:**

The `HAClient` protocol needs to grow an `AsyncStream`-based event interface alongside `fetchHome()`, or replace it. The brief already shows the target shape (Section 14.1):

```swift
actor HAClient {
    func callService(domain: String, service: String, data: [String: Any]) async throws
    var stateChanges: AsyncStream<StateChangeEvent> { get }
    @Published var isConnected: Bool
}
```

`HAController` becomes a subscriber to `stateChanges` rather than a poller. The mock implementation can feed the stream from a timer, making tests identical.

**The actor migration (Phase 0d.1):** The brief and the todos both flag this — `HAClient` should migrate from `final class` to `actor` before WebSocket lands. This is the right sequence: actor migration first (0d.1), REST writes second (0d.2), WebSocket third (0d.3). Don't attempt WebSocket on a non-actor client — the concurrency model will be wrong from the start.

**Recommendation:** Before Phase 0d.1, design the `AsyncStream`-based `HAClient` protocol shape in full, including `callService`. Lock the interface before migrating the existing class to actor — doing both at once is risky.

---

## 2. The HomeSnapshot Pattern — Does It Survive WebSocket?

### Current shape

Every 10 seconds: fetch all entities → parse into `HomeSnapshot` → `applySnapshot()` overwrites the entire home state in one `withAnimation` block. Simple, correct, stateless.

### What WebSocket changes

WebSocket pushes `StateChangeEvent` records: one entity changed, here's its new state. This is a fundamentally different model — delta-based, not full-replacement.

**The core tension:** The current `HomeSnapshot` → full overwrite pattern works cleanly with REST because full state arrives all at once. With WebSocket, we'd receive events like "climate.downstairs hvac_action changed from idle to heating." To update `home.climate.activity`, the controller needs to:

1. Receive the event
2. Identify which part of `HomeState` it maps to
3. Apply a targeted mutation

This is reconciliation logic — the controller decides how an entity event maps to the view model. It's not trivial, and it's not what `applySnapshot()` does today.

**Option A: Keep HomeSnapshot, batch WebSocket events**  
Buffer incoming events, apply them to a running `HomeState` copy, emit a synthetic `HomeSnapshot` on a short timer (e.g., 100ms debounce). Preserves the current `applySnapshot()` pattern. Downside: you lose the sub-100ms latency benefit of WebSocket.

**Option B: Delta reconciliation in HAController**  
`HAController` maintains `home` as mutable state and applies per-entity mutations as events arrive. `HomeSnapshot` becomes an initial-fetch mechanism only (used for startup, not steady-state). Downside: reconciliation logic grows complex as entities multiply.

**My recommendation: Option A for Phase 0d, revisit at Phase 1.**  
The debounce approach preserves the current architecture while allowing WebSocket to replace the polling loop. The latency you lose (100ms batch window vs. true real-time) is invisible for temperature readings, light states, and lock states — all of which change slowly. True real-time matters for the voice layer (speaking a command → seeing the light change instantly) — design the delta path then, when the exact requirements are known.

**Impact on Phase 0c.10 work:**  
The `PanelStateController` / `PhotoController` / `HAController` split is correct regardless of which option we pick. `HAController` is the right place for reconciliation logic either way.

---

## 3. State Management at Scale — Controllers, Voice, Alerts, Shopping

### The coordinator pattern is correct

The thin `AppModel` coordinator + focused controllers is the right model for this app. It's not a full Redux/TCA architecture (which would be over-engineered) and it's not a god-object monolith (which is where we started). It's the sweet spot for a 5-screen app with well-defined boundaries.

### What voice and alerts become

**Voice (Phase 2):** A `VoiceController`. It owns:
- `AVAudioEngine` capture session
- Apple Speech framework session
- Transcript state (`userUtterance`, `claudeResponse`)
- Voice state machine: `.idle` → `.listening` → `.thinking` → `.speaking` → `.idle`
- Claude API session (with HA tool definitions)
- TTS playback

`AppModel` grows `let voiceController = VoiceController()` and forwarded properties. The view layer (`VoiceOverlay`, phase 2) binds to `model.voiceState`, `model.userUtterance`, etc. This fits cleanly into the current pattern.

**Alerts (Phase 1):** An `AlertController`. It owns:
- `[Alert]` queue
- The 30-second dismiss timer
- Alert source attribution logic

HA state changes that trigger alerts — door/garage events — come through `HAController` or the WebSocket stream. `AlertController` subscribes (via a delegate or Combine subject — TBD when WebSocket shape is known). It maintains `var activeAlert: Alert?` which drives the `AlertOverlay` view.

**Shopping list, activity log (Phase 1):** Less clear. These aren't controllers in the same sense — they're more like data sources with a read-only view. Possible options:
- Properties on `HAController` (if data comes from HA)
- A dedicated `DataController` that owns shopping + activity (two related read-only data sources)
- Deferred until the actual data source is confirmed (iOS Reminders, HA `todo.shopping_list`, etc.)

### Write queue and rollback (Phase 0d.4)

This doesn't exist yet, but here's the shape I'd expect:

```swift
// In HAController (or a new WriteController)
var pendingWrites: [HAWrite] = []

func write(_ write: HAWrite) async throws {
    // Optimistic update
    applyOptimistic(write)
    pendingWrites.append(write)
    do {
        try await haClient.callService(write.domain, write.service, write.data)
        pendingWrites.removeAll { $0.id == write.id }
    } catch {
        // Rollback
        revertOptimistic(write)
        pendingWrites.removeAll { $0.id == write.id }
        throw error
    }
}
```

The rollback complexity grows with the number of entities that can be written. Phase 0d.2 ("fire and let next poll catch up") avoids rollback entirely — a design discipline worth preserving as long as possible. Rollback complexity is real; don't add it until the user experience demonstrably needs it.

---

## 4. The View Hierarchy at Full Scope — PanelState Evolution

### Current: three states

```swift
enum PanelState {
    case ambient
    case quickControl
    case fullControl
}
```

### What's coming

**Alert mode** (Phase 1): Must work from any panel state. The alert visual layer (radial wash, center card) overlays whatever state is active — it doesn't replace it. Alert is not a `PanelState` value; it's an overlay controlled by `AlertController.activeAlert`.

**Voice mode** (Phase 2): Same pattern — voice is a cross-cutting overlay, not a panel state. The voice animation layer (listening rings, transcript) floats above whatever state is active. `VoiceController.voiceState` drives the overlay, not `PanelState`.

**Shopping list, activity log** (Phase 1): These are drill-ins from full control. They're standard iOS sheets — they don't change `panelState`. No extension needed.

**Conclusion: `PanelState` stays as-is.** The ambient/quickControl/fullControl enum is correct and sufficient. Alerts and voice are orthogonal overlays. The discipline is to keep it that way — resist the temptation to put alert or voice into `PanelState` because they seem like "modes." They're not panel state; they're controller state.

**One thing to watch:** the current `onChange(of: model.panelState) { dismiss drillInRoom }` pattern in three places. When panels multiply (3+ targets), this pattern needs to be centralized — probably in a `dismissAllDrillIns()` method on `AppModel` that each controller calls during panel state transitions.

---

## 5. Multi-Target Architecture — When Does the Decision Happen?

### Current: single target, hardcoded first-floor config

`WallPanel` has one app target. Entity IDs and per-floor config are in `HAConfig` (UserDefaults). There's no mechanism for a second panel to have different config at compile time.

### The brief's target vision

```
HomeControl.xcworkspace
├── HomeControlCore (Swift package)
├── WallPanel (iPadOS 17+)
├── Personal (iPadOS 17+)
└── Phone (future)
```

The brief says per-device variation is compile-time config, not runtime UI (Principle 9). This means different Xcode targets with different configs baked in, not a settings screen.

### When does this matter?

**Phase 1:** Mount iPad 8 on 2nd floor. If the 2nd-floor panel needs different quick actions (TBD by Todd), it needs its own compile-time config. Two options:
- **Same target, different scheme:** Each scheme sets compiler flags or a build config that picks a `PanelConfig`. Simple, no new targets.
- **Separate Xcode target:** Clean separation, independent code signing, independent entitlements. More overhead now, correct structure forever.

**Phase 3:** Personal iPad Air gets its own target (no ambient, launches directly to full control). This is a genuine new target — different app behavior, not just different config.

**Recommendation: separate schemes for wall panels, new target for Personal.**
- `WallPanel-1stFloor`, `WallPanel-2ndFloor` schemes within the single `WallPanel` target, each pointing to a different `PanelConfig` constant. Schemes are cheap.
- `Personal` as a new Xcode target when Phase 3 begins. Not before — the overhead isn't justified yet.

**The `PanelConfig` struct doesn't exist yet.** `HAConfig` stores per-device entity IDs, but they're not compile-time — they're runtime-mutable from Settings. For a true compile-time config, we need a `PanelConfig` enum or struct that's baked in at build time. This should be designed before Phase 1 goes to a second device, not after.

**The decision timeline:** Before Phase 1 (2nd floor mount). That's the next meaningful target. Plan the scheme-per-panel approach then.

---

## 6. Testing Strategy — The Minimum Surface Worth Having

There are currently zero tests. With the controller split from Phase 0c.10, here's the minimum test surface that would prevent the most regressions:

**`PanelStateControllerTests`:** 
- State transitions: ambient → quickControl → fullControl → ambient
- Timeout fires: openQuickControl(), wait 15s, assert panelState == .ambient
- Timeout cancel: open, interact (openFullControl), assert quick timeout didn't fire
- These are pure logic tests, no UI, no async network. Fast and deterministic.

**`HAConfigTests`:**
- Keychain round-trip: save token, reload config, assert token persists
- Token migration: write old-format JSON to UserDefaults, load config, assert token moved to Keychain and removed from defaults
- Entity ID persistence: save config with custom entity IDs, reload, assert IDs preserved

**`DrillSourceTests`:**
- Not valuable — the enum is trivial. The compiler enforces completeness.

**`ClimateStateTests`** (existing model in HomeControlCore):
- `effectiveMode` when isEco is true
- `displayTargetTemp` in all four modes including eco with nil ecoFloor
- `ecoActiveSide` at midpoint boundary
- These are pure value-type computations — fast, no I/O.

**What I'd skip:** UI tests. The app is a live data panel — the only meaningful UI test is "does the panel look right at 10 feet." That's an eyes-on-device test, not a simulator test. The time investment in XCUITest infrastructure is not justified at this project's scale and team size.

**The right moment to add tests:** When Phase 0d.2 writes land. Writing to HA with optimistic updates is the first place where a regression would silently corrupt the user's home state (lights staying in wrong position, thermostat set incorrectly). That's when the test suite pays for itself.

---

## 7. Things Keeping Me Up at Night

**A. The "pane of glass" principle (Principle 14) is correct but discipline-dependent.**

The panel should pull from sources, push commands, and never compute state. But the current `ClimateState` already has computed properties: `effectiveMode`, `displayTargetTemp`, `ecoActiveSide`. These are view-model transformations (raw HA state → display-ready values), not policy decisions — so they're arguably fine. But the line between "view model transformation" and "computing state" is subtle, and it's easy to cross it without noticing.

The architectural guardrail: any new `var` on `ClimateState`, `HomeState`, or any model type should pass the test "is this a display transformation or a policy decision?" Policy decisions belong in HA automations. The panel should not contain logic that says "if floor-to-ceiling delta > 3°, surface the fan control" — that decision belongs in HA, and the panel should only surface the control when HA's input_select says to.

**B. The relationship between `HAController` and `HomeState` is becoming a write target, and the current model doesn't support that.**

Right now `home: HomeState` is set once per poll via `applySnapshot()`. In Phase 0d, writes happen: the user changes the thermostat, we call HA, we optimistically update `home.climate`. But `home` is an `@Observable` stored property on `HAController` — individual sub-mutations (`home.climate.activity = .heating`) will correctly trigger observation. The question is: do we want callers of `HAController` mutating `home.climate` directly, or through a controlled interface?

Controlled interface is safer. When writes land, `HAController.setThermostat(target: 68)` should:
1. Validate the write
2. Enqueue it
3. Apply optimistic update to `home.climate.targetTemp`
4. Return (or throw on failure)

Not: let the view layer mutate `home.climate.targetTemp` directly. This discipline is worth establishing before writes land, not after.

**C. The voice layer is a character, not a feature.**

This isn't an architecture concern exactly, but it's the one thing that could make or break the panel's identity. Principle 11 says "from the future." Section 11 of the brief describes the voice character in detail — dry, capable, Jarvis-adjacent. The architectural decision that matters is: when Phase 1.5 ships the canned-phrase button, every phrase in that bank becomes a canonical sample of what this voice sounds like. Those phrases need to be written by someone who has read Section 11 carefully, preferably out loud in the actual room.

The technical architecture for Phase 1.5 is simple (local audio files, `AVAudioPlayer`). The hard part is the writing. Don't let the ease of the technical implementation cause the writing to be underinvested.

**D. The five-target future is closer than it looks.**

By Phase 3, we have: 1st floor wall, 2nd floor wall, Personal iPad Air, maybe 3rd floor. Each device has its own entity IDs, its own behavior profile (ambient vs. no-ambient), and potentially its own quick actions. The single `HAConfig` in UserDefaults is already not the right structure for this — it's one flat config rather than per-device typed configs.

Before Phase 3, the right move is to design `PanelConfig` as a compile-time constant (a Swift enum or static struct, not a UserDefaults key). Entity IDs, floor name, quick actions, whether ambient mode exists — all baked in at build time per scheme. The brief already describes this (Section 13, Principle 9). The groundwork needs to be laid before the third device, not discovered when we're trying to ship it.

**E. The stacking polling loop is the first real safety concern in the codebase.**

Every call to `haController.startPolling()` (which is called from `saveHAConfig()`) spawns a new infinite `Task`. Multiple taps of "Save & Connect" in Settings → multiple polling loops → multiple calls to `applySnapshot()` on overlapping timers → possible race conditions on `home` state. It's in `HAController` now, clearly visible, isolated. Phase 0c.11 should fix it immediately before Phase 0d starts — it's the most likely source of subtle concurrency bugs when writes land.

**F. Accessibility is a first-class constraint, not a later consideration.**

Todd is deaf. The voice transcript is explicitly called out as "co-primary with audio" (brief §11.8). But there are other accessibility concerns in the current panel that haven't been addressed:

- `contentTransition(.numericText())` helps sighted users read changing temperatures. For VoiceOver users, the changing value needs an `accessibilityLabel` that describes the full state ("73 degrees, heating toward 68").
- The HVAC arrow encodes activity (↗ = heating, ↘ = cooling, ↔ = idle). VoiceOver cannot read a Canvas drawing — the arrow view needs an `accessibilityLabel`.
- The alert mode pulse (radial amber wash) is visible but has no accessible counterpart. The center card text provides the semantic content, but the glow is pure visual.

None of this is blocking for Phase 0c/0d, but it should be on the Phase 1 list when "real people in the house" usage begins. Todd's personal use is the test environment right now, but Mei and Cora use the panel too.

---

## Summary: The Decisions That Need to Happen Before Feature Stacking

1. **Before Phase 0d.1:** Design the `AsyncStream`-based `HAClient` protocol shape. Lock the interface. Then migrate to actor.

2. **Before Phase 0d.2 (writes):** Decide whether writes live in `HAController` or a new `WriteController`. Design the optimistic update + rollback pattern. Add `PanelStateControllerTests` and `HAConfigTests` — writes are the first place regressions would harm real home state.

3. **Before Phase 1 (second device mount):** Design `PanelConfig` as a compile-time constant per scheme. The current `HAConfig` in UserDefaults is not the right structure for multi-device.

4. **Before Phase 2 (voice):** Phase 1.5 canned phrases are the canonical character samples. Write them carefully, hear them in the room, revise. The TTS voice decision and phrase bank define what the voice character becomes.

5. **Ongoing discipline:** Honor Principle 14 (pane of glass). Every new computed property on a model type should be a display transformation, not a policy decision. Policy lives in HA.

---

*Written by Claude Code (Anthropic) at end of Phase 0c.10. The assessment reflects the codebase at v0.4.0 and the brief as updated 2026-04-26 (17 principles). DC and Todd should decide what becomes work.*
