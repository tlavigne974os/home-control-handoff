# Phase 0c.11 — Bug Consolidation + Quick Control Redefinition

**Tag:** v0.4.1  
**Commits:** 8191d2c, 1edef65, 3ce68bf, 01b661e, 6432d69, ecdcfde  
**Date:** 2026-04-26  
**Deployed:** pending

---

## What Was Built

Six-item pass: five targeted bug fixes and one behavioral redefinition of quick control. All items committed separately; zero behavior change outside the six scoped items.

---

### Item 1: Fix Stacking Polling Loop

**Before:** `startPolling()` spawned an untracked `Task { ... while true ... }`. Every call to `saveHAConfig()` stacked a new polling loop alongside the previous one. After two config saves, two concurrent fetch loops ran simultaneously.

**After:** `HAController` stores `private var pollingTask: Task<Void, Never>?`. `startPolling()` calls `pollingTask?.cancel()` before assigning a new task. `stopPolling()` added for symmetry. Same pattern applied to `weatherTask`. The `while true` replaced with `while !Task.isCancelled` so the check is explicit.

---

### Item 2: Entity-Level Health in HA Diagnostic

**Before:** Settings showed "Live ✓" whenever the HTTP connection succeeded, even if several of the configured entity IDs returned nothing from HA (misconfigured names, missing helpers, etc.).

**After:**

**`PollingHAClient`** computes entity health in `fetchHome()` after `fetchAllStates()` returns:
```swift
let configuredIds: [String] = [config.climateEntity, config.frontDoorEntity,
                                config.garageEntity, config.homeModeEntity]
    + Array(config.lightEntities.values)
let resolvedCount = configuredIds.filter { states[$0] != nil }.count
entityHealth = (resolved: resolvedCount, total: configuredIds.count)
```

**`HAController`** reads `entityHealthCount` after each successful fetch by casting `haClient as? PollingHAClient`. Exposed as an observable property forwarded through `AppModel`.

**`SettingsView`** shows:
- `"Live ✓"` (green) when all entities resolved
- `"Partial — X of Y"` (orange) when at least one entity was missing

Total entity count for default config: 8 (climate, frontDoor, garage, homeMode, 4 light groups).

---

### Item 3: Photo Swipes Instant

**Before:** `AmbientView` used `.transition(.opacity.animation(.easeInOut(duration: 1.75)))` — a pinned animation attached to the transition modifier. Pinned animations override the current transaction, so `Transaction.disablesAnimations = true` in `goNextPhoto()` / `goPrevPhoto()` had no effect. Manual swipes crossfaded just like auto-rotation.

**After:** Changed to `.transition(.opacity)`. The animation for the transition now comes from the calling transaction:
- `goNextPhoto()` / `goPrevPhoto()`: use `Transaction.disablesAnimations = true` → instant cut
- Auto-rotation (`advancePhoto(animated: true)`): uses `withAnimation(.easeInOut(duration: 1.75))` → 1.75s crossfade

`PhotoController` already had this split correctly — the bug was entirely in the view's transition modifier.

---

### Item 4: Timeout Resets on Interaction

**Before:** Timeout started when the panel opened and ran to zero regardless of subsequent touches. A user who spent 14 seconds in quick control and then tapped a tile would still get dismissed 1 second later.

**After:** `PanelStateController.recordInteraction()` cancels and reschedules the appropriate timeout based on current `panelState`:
```swift
func recordInteraction() {
    switch panelState {
    case .ambient:      break
    case .quickControl: cancelTimeouts(); scheduleQuickControlTimeout()
    case .fullControl:  cancelTimeouts(); scheduleFullControlTimeout()
    }
}
```

Wired via `.simultaneousGesture(DragGesture(minimumDistance: 0).onChanged { _ in model.recordInteraction() })` on the ZStack containing QuickControlContent and FullControlContent. Fires alongside existing button/drag handlers without interfering with them. Forwarded through `AppModel` as `func recordInteraction()`.

---

### Item 5: Version Label from Bundle

**Before:** `LabeledContent("Version", value: "0.3.0-phase0c")` — hardcoded string that required a manual edit on every build.

**After:** Reads `CFBundleShortVersionString` and `CFBundleVersion` from `Bundle.main`. Added `appVersion` and `appBuild` computed properties to `SettingsView`. "Phase" label remains static (it's meaningful context, not a version number).

---

### Item 6: Quick Control Redefinition

**Removed entirely from quick control:**
- `ClimateStripRow` — climate strip with mode pills and +/- buttons
- `ZoneTilesRow` — zone mini-tiles with drill-in sheet
- `QuickActionsRow` — compact pill tiles (now replaced by word tiles)
- `LightDrillInView` sheet — no longer reachable from quick control
- "more controls →" escalator hint

**New layout — two columns:**

**Left — STATUS:**
- `TrinityStatusBar` with SYSTEM → TARGET · HERE labels
- Horizontal divider
- OUTSIDE section: outdoor temp + condition string + precipitation if present

**Right — SECURITY + DOWNSTAIRS:**
- Front Door word tile — amber/alert styling when unlocked
- Garage word tile — amber/alert styling when open
- Spacer
- DOWNSTAIRS brightness slider — reads `rooms.first { $0.id == "downstairs" }?.brightness`; drag fires write-disabled toast and snaps back (Phase 0c writes disabled)

**Rationale:** Quick control is a glance + the two most-acted-on security tiles. Mode pills and zone tiles require deliberate engagement that belongs in full control. The slider provides affordance for the single most-used light without opening full control for a brightness tweak.

Also fixed: `RootView` simulator-only `LightDrillInView(roomId: sel.id, model:)` call site — missed during Phase 0c.10's DrillSource migration. Updated to `LightDrillInView(source: .room(id: sel.id), model:)`. Caught by the build.

---

## What Was Not Changed

- No WebSocket work (Phase 0d)
- No actual write capability (HAClient is still read-only)
- No new tests
- No model-layer changes

---

## Autonomous Decisions

**`recordInteraction()` via DragGesture(minimumDistance: 0) not TapGesture:** `TapGesture.onEnded` fires after lift, not on touch-down. `DragGesture(minimumDistance: 0).onChanged` fires immediately on contact — correct behavior for a timeout reset. A user who touches a slider but doesn't lift their finger should still reset the timeout.

**Entity health reads `haClient as? PollingHAClient` not a protocol method:** Adding `entityHealth` to the `HAClient` protocol would require `MockHAClient` to implement it (always returning `(8, 8)` or similar). The cast is appropriate here — `MockHAClient` can't have partial entity health by definition, and the diagnostic is only meaningful for `PollingHAClient`.

**Slider uses local draft state, not `.constant()`:** `.constant(brightness)` would make the slider non-interactive (thumb frozen). Using a draft binding allows the thumb to move during drag, providing physical feedback, while snapping back when editing ends and the toast fires. More honest than a frozen slider.

**`QuickActionsRow` not deleted from `BottomActionRow.swift`:** It's no longer used by anything, but removing it is a separate cleanup. It's dead code, not a bug. Flagged for Phase 0c.12 or whenever `BottomActionRow.swift` next changes.

---

## Things That May Need DC Evaluation

1. **Weather not available in quick control left panel:** If `weatherReady == false`, the left panel shows only the TrinityStatusBar with no second section. Layout is still correct (Spacer fills the gap), but the panel may look sparse until weather arrives on first launch (~2s). Acceptable for now.

2. **Slider brightness label shows 0% when room is off:** `brightness` is `nil` when the room is off (not 0.0). The slider reads `brightness ?? 0`, so it shows 0% when off. This is technically correct — the room has no brightness when off. Phase 0d can show "OFF" instead of "0%" when `room.brightness == nil`.

3. **`QuickActionsRow` is dead code in `BottomActionRow.swift`:** Still compiled, no longer called. Non-urgent.
