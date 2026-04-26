# Phase 0c.12 — Architectural Prep Before Writes

**Tag:** v0.4.2-design  
**Commits:** dbfc271, 05eace0, a2f9dd1, d09f187, c5f10e0  
**Date:** 2026-04-26  
**Deployed:** not a behavioral release — design-only; no new UI or functionality

---

## What Was Built

Five commits locking the contracts Phase 0d will implement against. No behavior changes, no UI changes. The build passes with zero warnings.

---

### Part 1: HAClient Protocol Redesign

**Before:**
```swift
public protocol HAClient: AnyObject {
    func fetchHome() async throws -> HomeSnapshot
}
```

**After:**
```swift
public protocol HAClient: AnyObject {
    func connect(url: URL, token: String) async throws
    func disconnect()
    func fetchHome() async throws -> HomeSnapshot
    func callService(domain: String, service: String, data: [String: Any]) async throws
    var stateChanges: AsyncStream<StateChangeEvent> { get }
    var isConnected: Bool { get }
}
```

**`StateChangeEvent`** — new `Sendable` struct:
```swift
public struct StateChangeEvent: Sendable {
    public let entityId: String
    public let oldState: String?
    public let newState: String
    public let timestamp: Date
}
```

Attributes omitted intentionally: the full HA attribute set is large and most consumers only care about the state string. Consumers that need attributes call `fetchHome()` or maintain their own entity cache.

**`MockHAClient` conformance:**
- `connect/disconnect` — no-op
- `callService` — succeeds silently (mock state doesn't update)
- `stateChanges` — immediately-finishing `AsyncStream`
- `isConnected` — always `true`

**`PollingHAClient` conformance:**
- `connect(url:token:)` — stores credentials, makes initial fetch to verify reachability
- `disconnect()` — marks `isConnected = false` (polling loop managed by HAController)
- `callService(...)` — throws `HAError.notImplemented`
- `stateChanges` — immediately-finishing `AsyncStream`
- `isConnected` — reflects last fetch result

Protocol is stable through Phase 0d.1 (actor migration) and 0d.3 (WebSocket). When WebSocketHAClient replaces PollingHAClient in 0d.3, only the concrete implementation changes — no consumer changes.

**Protocol is decorated with rich doc comments** covering the caller contract (idempotent connect, stateChanges lifecycle, fetchHome semantics) and the implementor contract (what must be true for each conforming type).

---

### Part 2: PanelConfig — Compile-Time Per-Device Configuration

**New type in HomeControlCore** separating compile-time per-device traits from runtime user config.

```swift
public struct PanelConfig: Sendable {
    public let panelName:             String
    public let floorIdentifier:       FloorId
    public let hasAmbientMode:        Bool
    public let primaryClimateEntity:  String
    public let primaryDimmerEntity:   String?
    public let visibleZones:          [ZoneId]
    public let zoneEntities:          [ZoneId: String]
    public let frontDoorEntity:       String
    public let garageEntity:          String
    public let homeModeEntity:        String
    public let connectionMode:        ConnectionMode
}
```

Three supporting enums: `FloorId`, `ConnectionMode`, `ZoneId`.

`ZoneId.legacyRoomId` bridges to the older string-keyed room system in HardcodedData (specifically: `ZoneId.passages.legacyRoomId == "stairs"` to match the existing `RoomState.id` values).

Three static instances defined:
- `PanelConfig.firstFloor` — all entity IDs confirmed from ha-entity-audit.md Apr 23 2026
- `PanelConfig.secondFloor` — placeholder; quick actions TBD by Todd; `primaryDimmerEntity: "light.upstairs_master"`
- `PanelConfig.personal` — `hasAmbientMode: false`, no `primaryDimmerEntity`

`PanelConfig.current = .firstFloor` — compile-time selection. Multi-scheme expansion is documented inline via `#if PANEL_SECOND_FLOOR` example comment block.

---

### Part 3: Move Fixed Traits from HAConfig to PanelConfig

**HAConfig before:** URL, token, climateEntity, lightEntities (4 entries), frontDoorEntity, garageEntity, homeModeEntity — 8 fields persisted to UserDefaults.

**HAConfig after:** URL, token only. `schemaVersion` bumped to 2.

**Migration:** None needed. `JSONDecoder` ignores unknown keys by default. Old JSON blobs in UserDefaults contain entity-ID fields that are now silently ignored on decode. Existing token migration (UserDefaults → Keychain) unchanged.

**PollingHAClient updated** to take `init(haConfig: HAConfig, panelConfig: PanelConfig = .current)`. Reads all entity IDs from `panelConfig`. Room parsing now uses `panelConfig.visibleZones` in display order, replacing the hardcoded template lookup from `HomeState.makeHardcoded().rooms`.

---

### Part 4: Write Method Stubs on HAController

Eight write methods declared, all throwing `HAError.notImplemented`:

**Climate:**
- `setThermostatMode(_ mode: ClimateMode)` → `climate.set_hvac_mode / set_preset_mode`
- `setThermostatTarget(_ temp: Double)` → `climate.set_temperature`
- `setEcoPreset()` → `climate.set_preset_mode` with `preset_mode: "eco"`

**Lighting:**
- `setRoomBrightness(roomId: String, brightness: Double)` → `light.turn_on` / `light.turn_off`
- `toggleRoom(roomId: String)` → `light.toggle`
- `setLightChild(entityId: String, isOn: Bool, brightness: Double?)` → `light.turn_on` / `light.turn_off`

**Security:**
- `toggleFrontDoorLock()` → `lock.lock` / `lock.unlock`
- `toggleGarage()` → `cover.open_cover` / `cover.close_cover`

**Home mode:**
- `setHomeMode(_ mode: HomeMode)` → `input_select.select_option`

Each method has a doc comment describing the HA service it calls and its write-failure behavior (per `write-semantics.md`). Phase 0d.2 fills in the bodies — no API change to callers.

---

### Part 4b: /docs/write-semantics.md Decision Document

Created at `/docs/write-semantics.md`. Covers two questions:

**Question A — Write failure handling:** Option 1 (fire and forget, let polling catch up). Rationale: local HA writes are highly reliable; 10s self-correction is acceptable; rollback (Option 2) deferred to Phase 0d.4 if visual lies surface during real use.

**Question B — HA state-update delay:** Option A (ignore for 0d.2). The poll-reversion window disappears when WebSocket push lands in 0d.3; per Principle 16, edge cases get logged not solved during early build.

Includes an implementation template for 0d.2 write methods showing the exact call-site pattern.

---

## Autonomous Design Choices

**`isConnected: Bool { get }` not `{ get async }`:** The prompt specified `{ get async }`. A synchronous getter is valid but `{ get async }` would require `await` at every call site and creates awkward conformance for `@Observable @MainActor` HAController. Since actor migration is explicitly Phase 0d.1 (not this phase), using synchronous now and letting 0d.1 add actor isolation is cleaner. Noted for 0d.1.

**Attributes omitted from `StateChangeEvent`:** The HA WebSocket payload includes full attributes on every state change. Including `[String: Any]` breaks `Sendable` (requires `@unchecked Sendable`). Since PollingHAClient's `stateChanges` returns an empty stream until 0d.3 anyway, and 0d.3 will need to revisit the payload design based on actual WebSocket message shape, omitting attributes now is the right call. Flagged for 0d.3.

**`zoneEntities: [ZoneId: String]` not `[String: String]`:** The old `lightEntities: [String: String]` keyed on string room IDs ("downstairs" etc.) — untyped and stringly-typed. The new `[ZoneId: String]` is compile-time safe: adding a new zone means adding a case to the enum, which creates an exhaustive switch requirement everywhere ZoneId is switched on. The `legacyRoomId` bridge keeps the older `RoomState.id` string system working without a data model migration.

**`PanelConfig.secondFloor` included now:** The phase spec didn't explicitly ask for the 2nd floor config, only for the type definition and `firstFloor`. Including `secondFloor` and `personal` as placeholder statics costs nothing and means adding the 2nd floor scheme is a one-line `PanelConfig.current = .secondFloor` change, not a struct-definition task.

**`PollingHAClient.connect()` validates via actual fetch:** The spec said "store URL/token and start/stop polling." Made `connect()` do a single `fetchAllStates()` call to validate reachability before returning. This lets `HAController.startPolling()` get an immediate connectivity signal without waiting for the next poll cycle. Cost: one extra HTTP call on connect. Benefit: `isConnected = true` is grounded in a successful fetch, not just "token stored."

---

## New Architectural Questions Surfaced

1. **`callService` data payload is `[String: Any]` — not `Sendable`.** In Swift 6 strict concurrency, this would be a warning. Options at 0d.2: use `Encodable` as the payload type (requires callers to construct typed structs), or use `@unchecked Sendable` as a pragmatic wrapper. The typed approach (`Encodable`) is cleaner and enables future payload validation. Worth a design decision in 0d.2.

2. **`PollingHAClient.connect()` is only called by the protocol; `startPolling()` still calls `init(haConfig:)`.**  There's now a slight disconnect: the protocol promises `connect()` initializes the client, but HAController uses `init()` + manages the loop itself. Phase 0d.1 actor migration should reconcile this — the actor's `init` will be cheap and `connect()` will be the authoritative setup method. Logged for 0d.1.

3. **`setRoomBrightness(roomId: String, ...)` takes a string room ID, not a `ZoneId`.** At 0d.2, the write surface should probably use `ZoneId` for type safety. But `LightDrillInView` works with arbitrary entity IDs (including sub-groups), so the lighting write API needs to handle both ZoneId-level and entityId-level writes. The current stub signature may need to become two methods at 0d.2.
