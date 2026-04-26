# Phase 0d.1 — HAClient Actor Migration
**Date:** 2026-04-26  
**Tag:** v0.4.3-actor  
**Build:** 0.4.3 / build 43  
**Status:** ✅ Complete — deployed to device, compiled clean under `-strict-concurrency=complete`

---

## Original DC Prompt

> **Phase 0d.1 — HAClient Actor Migration**
>
> Goal: promote HAClient and its two concrete types from AnyObject-constrained classes to Swift actors. No UI changes. No behavior changes. This is pure architecture — tighten the concurrency model so strict checking can pass.
>
> **Part 1 — HAClient protocol: AnyObject → Actor**
> Change `public protocol HAClient: AnyObject` to `public protocol HAClient: Actor`. This makes all protocol members actor-isolated. Update the comment block. Note: callers in HAController (@MainActor) will now need `await` on every haClient.* call — that's expected and correct.
>
> **Part 2 — PollingHAClient: class → actor**
> Change `final class PollingHAClient: HAClient` to `actor PollingHAClient: HAClient`. Remove `pollingTask` and the poll loop — HAController already owns loop scheduling. Reconcile `connect()` vs `init()`: `connect()` should validate credentials via one `fetchAllStates()` call and throw on failure. `disconnect()` marks `isConnected = false`. `fetchHome()` is called by HAController's loop, not internally.
>
> **Part 3 — MockHAClient: class → actor**
> Change `public final class MockHAClient: HAClient` to `public actor MockHAClient: HAClient`. Stubs unchanged.
>
> **Part 4 — entityHealth on the protocol (Option A)**
> Add `var entityHealth: EntityHealth? { get }` to the HAClient protocol. Define `public struct EntityHealth: Sendable { let resolved: Int; let total: Int; var isFullyResolved: Bool }`. MockHAClient returns `EntityHealth(resolved: 8, total: 8)`. PollingHAClient computes it in `fetchHome()`. HAController reads `await haClient.entityHealth` — no cast to PollingHAClient anywhere. Update AppModel and SettingsView (`!h.isFullyResolved`).
>
> **Part 5 — callService: [String: Any] → [String: any Sendable]**
> HA service data values are always String/Int/Double/Bool — all Sendable. Change the signature to `data: [String: any Sendable]` throughout (protocol + both implementations + HAController stubs).
>
> **Part 6 — HomeSnapshot: Sendable + cascade**
> Mark `HomeSnapshot: Sendable`. This cascades: add `: Sendable` to all value types in HomeModels.swift that HomeSnapshot (transitively) contains. All are pure structs/enums — unconditional conformance, no `@unchecked`.
>
> **Part 7 — Strict concurrency on HomeControlCore**
> In Package.swift, add `.unsafeFlags(["-strict-concurrency=complete"])` to the HomeControlCore target. Build must succeed with zero warnings. Fix anything the checker surfaces.
>
> **Suggested commit sequence:**
> 1. Protocol AnyObject→Actor + StateChangeEvent Sendable verification + EntityHealth  
> 2. PollingHAClient class→actor + reconcile connect()/init()/polling loop  
> 3. MockHAClient class→actor  
> 4. Caller updates (HAController, any other consumers)  
> 5. entityHealth migration  
> 6. Strict concurrency enablement + any cleanups  
>
> Tag the final state `v0.4.3-actor`.

---

## What Changed

Phase 0d.1 is a pure architecture pass: no UI changed, no behavior changed. The goal was to promote `HAClient` and its concrete implementations from `AnyObject`-constrained classes to Swift actors, enabling safe data-race detection under the complete concurrency checker.

### 1. HAClient Protocol: `AnyObject` → `Actor`

```swift
// Before
public protocol HAClient: AnyObject { ... }

// After
public protocol HAClient: Actor { ... }
```

Actor protocol conformance means all protocol members are actor-isolated. Callers in `HAController` (`@MainActor`) cross the actor boundary with explicit `await` on every `haClient.*` call.

### 2. EntityHealth on the Protocol (Option A)

The old pattern cast `haClient as? PollingHAClient` to read entity health — fragile and wrong. Option A was chosen: add `entityHealth: EntityHealth?` directly to the protocol.

```swift
public struct EntityHealth: Sendable {
    public let resolved: Int
    public let total: Int
    public var isFullyResolved: Bool { resolved == total }
}

// In protocol:
var entityHealth: EntityHealth? { get }
```

`MockHAClient` returns `EntityHealth(resolved: 8, total: 8)` (always healthy). `PollingHAClient` computes it from configured entity IDs vs. resolved states on each `fetchHome()`. `HAController.fetchHA()` reads it with `await haClient.entityHealth` — no cast anywhere.

### 3. PollingHAClient: `final class` → `actor`

```swift
// Before
final class PollingHAClient: HAClient { ... }

// After
actor PollingHAClient: HAClient { ... }
```

Key reconciliation: the polling loop was removed from `PollingHAClient`. `HAController` owns the `Task` (scheduling is a controller concern). `PollingHAClient` only knows how to execute a single `fetchHome()` or `connect()` call. `connect()` validates credentials via one `fetchAllStates()` call and throws fast on bad URL or invalid token.

### 4. MockHAClient: `final class` → `actor`

```swift
// Before
public final class MockHAClient: HAClient { ... }

// After  
public actor MockHAClient: HAClient { ... }
```

### 5. callService Signature: `[String: Any]` → `[String: any Sendable]`

`[String: Any]` is not `Sendable` and cannot cross actor boundaries safely. HA service data values are always `String`, `Int`, `Double`, or `Bool` — all `Sendable`. Clean fix with no loss of expressiveness.

### 6. HomeSnapshot: Sendable

```swift
public struct HomeSnapshot: Sendable { ... }
```

Required for safe crossing from the `HAClient` actor to `HAController` (@MainActor). This cascaded to requiring `Sendable` on all value types in `HomeModels.swift`.

### 7. Sendable on All HomeControlCore Value Types

Added `: Sendable` to: `ClimateMode`, `ClimateActivity`, `EcoSide`, `ClimateState`, `LightChild`, `RoomState`, `PrecipType`, `PrecipitationForecast`, `WeatherState`, `DoorState`, `GarageState`, `HouseStatus`, `HomeMode`, `ShoppingItem`, `ActivityEvent`, `ActivityTag`.

All are pure value types (structs/enums) with no reference-type stored properties, so `Sendable` conformance is unconditional.

### 8. Strict Concurrency Checking Enabled

```swift
// Package.swift
.unsafeFlags(["-strict-concurrency=complete"])
```

Build succeeded with **zero warnings** under the complete checker. This is the Phase 0d.1 headline achievement.

---

## Commit Sequence

| # | SHA | Message |
|---|-----|---------|
| 1 | dd4189b | actor: promote HAClient protocol from AnyObject to Actor |
| 2 | 11aa523 | actor: PollingHAClient class → actor + reconcile connect/init/loop |
| 3 | 336354a | actor: MockHAClient class → actor |
| 4 | 4ad4c48 | actor: update callers for actor-isolated HAClient |
| 5 | 1b0d69e | actor: Sendable conformance on all HomeControlCore value types |
| 6 | 8a139f8 | actor: enable strict concurrency checking on HomeControlCore |

Tagged: `v0.4.3-actor`

---

## HAController Polling Pattern (Final)

```swift
func startPolling() {
    pollingTask?.cancel()
    let newClient: any HAClient = haConfig.isConfigured
        ? PollingHAClient(haConfig: haConfig) : MockHAClient()
    haClient = newClient

    pollingTask = Task {
        if haConfig.isConfigured, let url = URL(string: haConfig.baseURL) {
            haConnectionState = .connecting
            do {
                try await newClient.connect(url: url, token: haConfig.token)
            } catch {
                haConnectionState = .error(error.localizedDescription)
                return
            }
        }
        while !Task.isCancelled {
            await fetchHA()
            try? await Task.sleep(for: .seconds(10))
        }
    }
}

private func fetchHA() async {
    haConnectionState = .connecting
    do {
        let snapshot = try await haClient.fetchHome()
        applySnapshot(snapshot)
        haConnectionState = .connected
        entityHealthCount = await haClient.entityHealth   // protocol, no cast
    } catch {
        haConnectionState = .error(error.localizedDescription)
    }
}
```

---

## Architecture Invariants Established

- **HAController owns the loop.** `PollingHAClient` executes individual calls; `HAController` schedules them.
- **connect() validates before the loop starts.** Bad credentials fail fast with a surfaced error. No silent misconfiguration.
- **No casts to concrete types anywhere.** Everything goes through the `HAClient: Actor` protocol.
- **Actor boundary crossings are all explicit.** Every `await haClient.*` call in `HAController` is a documented, intentional crossing.
- **Strict concurrency is a compile-time gate.** Any future PR that introduces a data race will not build.

---

## Next Phase

**Phase 0d.2 — Write Implementation**

Implement the 8 write method stubs in `HAController` (all currently throw `.notImplemented`):
- `setThermostatMode`, `setThermostatTarget`, `setEcoPreset` (climate)
- `setRoomBrightness`, `toggleRoom`, `setLightChild` (lighting)
- `toggleFrontDoorLock`, `toggleGarage` (security)
- `setHomeMode` (home mode)

Failure semantics: Option 1 fire-and-forget (per `docs/write-semantics.md`). Polling corrects within 10s on failure.

`PollingHAClient.callService()` will be implemented as a `POST /api/services/{domain}/{service}`.
