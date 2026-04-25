# Phase 0c.9 — Code Health Audit

**Date:** 2026-04-25  
**Scope:** WallPanel + HomeControlCore  
**Note:** Audit only. Nothing was changed.

---

## 1. Components That Have Gotten Complicated

**`MainPanelView.swift` — 700+ lines, does everything**

This file started as a panel host and has absorbed: a custom Canvas shape renderer (`HVACArrowView`), three file-level helper functions (`hvacArrowConfig`, `hvacArrowKey`, plus `sectionLabel`), twelve private view structs (StatusBarRow, AlertGlowIndicator, TrinityStatusBar, QuickControlContent, FullControlContent, ClimateSection, RoomsSection, RoomTile, BottomActionRow, ClimateStripRow, ZoneTilesRow, ZoneMiniTile, QuickActionsRow), and the `HVACArrowStyle` enum. The file is coherent in topic but far too large to navigate or reason about as a unit. The natural splits are obvious: `HVACArrow.swift`, `StatusBarRow.swift`, `ClimateSection.swift`, `RoomsSection.swift`. None of these need to wait for a feature.

**`StatusBarRow` — doing six things**

This single view manages: its own `Timer.publish` clock, the ambient vs close-up dual-layout crossfade, gesture recognition (competing with the identical gesture in `RootView`), ambient tap-to-open, arrow rendering and animation, and house indicator button wrapping. The gesture duplication between `StatusBarRow` and `RootView` is particularly confusing — there are two `DragGesture` recognizers for basically the same intent.

**`LightDrillInView` — ad-hoc tagged union in disguise**

The recursive drill-in refactor introduced three optional stored properties (`roomId: String?`, `groupName: String?`, `groupChildren: [LightChild]?`) with computed properties to resolve them at runtime. This is a manually-implemented tagged union. Swift has an actual tagged union: 

```swift
enum DrillSource {
    case room(id: String)
    case group(name: String, children: [LightChild])
}
```

The current approach is not wrong, but a future reader has to understand the invariant ("exactly one of roomId or groupName is non-nil") to work safely with it. Flagged as something to fix before the code has a second author.

**`ClimateSection.trinityCard` — conditional layout inside a ViewBuilder**

The trinity card in full control has an `if climate.effectiveMode == .off { ... } else if let arrow { ... }` branch that returns entirely different view trees. This is fine SwiftUI but the two paths are long enough that they're hard to read inline. Extracting `OffTrinityCard` and `ActiveTrinityCard` would make this clearer.

---

## 2. State Management

**The polling loop is not cancellable and stacks on re-entry.**

`startHA()` spawns `Task { await fetchHA(); while true { ... } }` with no stored reference. The Task is not cancelled before the next call. `saveHAConfig()` calls `startHA()`, which means every time the user taps "Save & Connect" in Settings, a new infinite polling loop starts — the old one keeps running alongside it. After three taps you have three loops all writing to `home` simultaneously. Writes are `@MainActor` so there's no data race, but the polling cadence is now 3x and old `haClient` references in the dead loops still hold a reference to whatever `PollingHAClient` was current at spawn time.

Same pattern exists for `startWeather()` and `startPhotos()` — but those are only called once from `RootView.onAppear`, so the re-entry risk doesn't apply in practice.

**`weatherReady` and `weatherError` duplicate state from `WeatherService`.**

`WeatherService.shared.lastError` exists. `AppModel.weatherError` mirrors it. The mirror is populated in `fetchWeather()`:

```swift
weatherError = WeatherService.shared.lastError
```

These can drift. If `WeatherService.fetch()` succeeds on the second attempt but the assignment is ordered wrong, `weatherError` could be stale. The right model is a single truth — either own the error in AppModel or own it in WeatherService, not both.

**`home: HomeState = .makeHardcoded()` is the live initial state.**

When the app launches with a working HA token, users see the hardcoded mock data (Upstairs 75%, Downstairs 60%, climate at 68°/71°/73°) until the first HA fetch completes — typically 1-3 seconds. This is brief but technically wrong: the display is showing false information confidently. The initial state should be a loading/pending state, not fabricated data. Practically invisible today with a fast local connection; will become visible if HA is slow or unreachable at launch.

**`targetTemp` was phantom state in the old `ThermostatRow`.**

Removed in 0c.7, but worth noting for the pattern: `@State private var targetTemp: Double = 68` lived in `QuickControlContent` and was initialized from `model.home.climate.targetTemp` in `.onAppear`. Any change to `model.home.climate.targetTemp` from a poll wouldn't update it (`.onAppear` fires once), and any local drag of the slider wouldn't update the model. It was decorative state. The current `ClimateStripRow` doesn't have this problem — it reads directly from `climate.targetTemp` — but the pattern should be watched for in Phase 0d when writes land.

---

## 3. Hardcoded Values That Should Be Tokens

All file:line references are approximate to the current build.

**`MainPanelView.swift` — design tokens missing:**

| Value | Where | Should be |
|---|---|---|
| `width: 42, height: 28` | TrinityStatusBar arrow | `HearthArrowSize.statusBar` |
| `width: 48, height: 32` | ClimateSection arrow | `HearthArrowSize.heroCard` |
| `width: 72, height: 48` | Ambient arrow | `HearthArrowSize.ambient` |
| `minWidth: 180` | Time block frame | unnamed constant |
| `UIScreen.main.bounds.width * 0.52` | ClimateSection frame | `HearthLayout.climateSectionFraction` |
| `15` (geistMono in trinityCol) | ClimateSection | should be `HearthTypeSize.labelMd` (which is 15 — so actually matches, but is a coincidental magic number rather than a named reference) |
| `32` (instrumentSerif in trinityCol) | ClimateSection | needs a name — `HearthTypeSize.trinityValue`? |
| `19` (geistMono "CLIMATE" label) | ClimateStripRow | needs a name — `HearthTypeSize.sectionLabel`? |
| `36` (instrumentSerif current temp) | ClimateStripRow | needs a name — `HearthTypeSize.stripTemp`? |
| `18` (instrumentSerifItalic target) | ClimateStripRow | needs a name |
| `height: 56` (Divider in trinity card) | ClimateSection | `HearthLayout.trinityCardHeight` |
| `h * 0.09` (arrow stroke weight) | HVACArrowView | internal constant in the struct |
| `h * 0.32` (arrowhead arm) | HVACArrowView | internal constant in the struct |
| `0.75` (alert glow pulse low) | AlertGlowIndicator | `HearthAnimation.alertGlowPulseLow` |
| `2.0` (alert pulse duration) | AlertGlowIndicator | `HearthAnimation.alertPulseDuration` |
| `width: 230` (time block ambient) | StatusBarRow.ambientContent | unnamed |

**`AppModel.swift`:**

| Value | Line | Should be |
|---|---|---|
| `1.5` seconds toast display | ~56 | `HearthAnimation.writeToastDuration` |
| `0.2 / 0.3` seconds toast animations | ~54, ~58 | `HearthAnimation.writeToastFadeIn/Out` |
| `10` seconds HA poll interval | ~243 | `HAConfig.pollInterval` or `HearthAnimation` |

**`SettingsView.swift`:**

- Line 107: `"0.3.0-phase0c"` — already stale (we're at v0.3.8). This should read from a `BuildInfo` constant or `Bundle.main.infoDictionary`. Right now it requires manual update on every release and is already wrong.

**`HAConfig.swift`:**

- `"http://homeassistant.local:8123"` — reasonable default but should be a named constant (`HAConfig.defaultBaseURL`) so it's findable.

**Opacity values repeated across files:**

`Color.hearthAmber.opacity(0.07)` for lit tile fill, `Color.hearthAmber.opacity(0.25)` for lit tile border, `Color.hearthAlert.opacity(0.06)` for alert fill, `Color.hearthAlert.opacity(0.4)` for alert border — these appear in `RoomTile`, `ZoneMiniTile`, `LightChildRow`, `BottomActionRow`, and anywhere a state-colored tile is drawn. They should be `Color.hearthAmber.tileFill` / `.tileBorder` style extensions, or at minimum named constants.

---

## 4. MockHAClient vs PollingHAClient — The "Connected" but Stale Data Issue

**What's happening when tiles show 100% but lights aren't actually on:**

`PollingHAClient.parseRooms()` uses hardcoded room template data as its scaffolding:

```swift
let template = HomeState.makeHardcoded().rooms  // preserves room order & IDs
return template.map { room in
    guard let s = states[room.entityId] else { return room }  // <-- KEY LINE
    ...
```

If a light entity isn't found in the HA response (wrong entity ID, entity unavailable, etc.), `return room` returns the **hardcoded room**, which has `brightness: 0.75` for Upstairs and `brightness: 0.60` for Downstairs. So you'd see tiles lit at those specific values, not at 100%.

If tiles show exactly 100%, that means HA **is** returning the entity, the state **is** "on", but `brightness` attribute is missing or zero — and the code defaults it:

```swift
let brightness = s.attributes.double("brightness").map { $0 / 255.0 }
return RoomState(
    brightness: isOn ? (brightness ?? 1.0) : nil,  // <-- 1.0 = 100% when brightness is nil
    ...
```

**This is the actual bug for 100% tiles when lights are off:** if the HA entity's state is "on" (some switches report "on" even at 0 brightness), the app shows 100% because the brightness attribute is absent or zero, and the fallback is `1.0`.

**The "Connected" check is correct but incomplete.** `haConnectionState = .connected` is set when the HTTP fetch of `/api/states` succeeds. It does not check whether individual entity IDs resolved correctly. There's no per-entity error surface. A bad entity ID silently falls back to hardcoded data while the diagnostic shows "Live ✓". This is a real UX trust issue.

**Protocol switching is clean but the timing is not.** When `startHA()` swaps `haClient`, any in-flight `fetchHome()` call on the old client is abandoned (the old Task context will finish writing to `home` even though `haClient` has been replaced). Because everything is `@MainActor` this won't corrupt state, but a stale result from the old client can briefly overwrite fresh results from the new client if they happen to complete in the wrong order.

---

## 5. Photo State Machine

**Why swipes might still be fading (not snapping):**

The intent is correct: `goNextPhoto()` calls `advancePhoto(animated: false)`, which wraps the state mutation in a `Transaction` with `disablesAnimations = true`. However, the `Image` view in `AmbientView` has:

```swift
.transition(.opacity.animation(.easeInOut(duration: HearthAnimation.photoCrossfadeDuration)))
.id(img)
```

The `.animation()` modifier embedded inside `.transition()` creates what SwiftUI calls a "pinned" or "attached" animation. In some SwiftUI versions, this animation **ignores** `Transaction.disablesAnimations = true` — the transition's own animation always fires. This is a documented-but-confusing SwiftUI behavior: an `.animation()` on a `.transition()` is evaluated at definition time, not transaction time.

If swipes are still fading, this is the cause. The fix is to remove the inline `.animation()` from the transition and let the surrounding transaction control it:

```swift
// Instead of:
.transition(.opacity.animation(.easeInOut(duration: ...)))
// Use:
.transition(.opacity)
// And drive animation from the caller's withAnimation/withTransaction
```

**The photo state machine is otherwise solid.** `ShuffleQueue` is well-isolated, history/back/advance are clean. The `startRotationLoop()` checks `panelState == .ambient` before advancing — correct. The `reloadAlbum` / `selectAlbum` / `advancePhoto` chain is reasonable.

**One structural issue:** `startRotationLoop()` is an infinite `Task { while true { } }` attached to nothing. It'll continue running even if the photo library is deauthorized mid-session. It checks `panelState == .ambient` but not whether `hasAlbumSelected` or `authorizationStatus == .authorized`. Benign in practice — `shuffleQueue` will be empty so `advance()` returns nil and nothing happens — but it's silent.

---

## 6. Token Persistence

**The token loss on rebuild is expected iOS behavior, not a bug.**

When Xcode installs a fresh build (not an update to an existing install), iOS deletes the app's data container, which includes `UserDefaults`. This is standard. It explains why the token disappears every time you rebuild from Xcode — a full reinstall, not an update.

**The storage approach has two real problems:**

1. **No `synchronize()` after save.** `HAConfig.save()` calls `UserDefaults.standard.set(data, forKey: key)` with no `synchronize()`. On modern iOS, UserDefaults flushes to disk asynchronously. If the app is killed by the OS immediately after saving (background memory pressure, Xcode stop button), the save may not have been written. For a token this is annoying — the user saved it, the app died, it's gone. `UserDefaults.standard.synchronize()` after every `HAConfig.save()` call is the low-effort fix.

2. **Plain UserDefaults for a credential.** The HA long-lived token is a sensitive credential — it provides full HA API access. Storing it in UserDefaults means it's readable by anyone with file system access to the app's container (not a concern on a locked iPad, but notable). Keychain is the correct storage for credentials and has the added benefit of surviving Xcode reinstalls (within the same developer team). The migration path is well-trodden.

**`PanelPreferences` is fine.** Album selection is not sensitive. The migration from legacy loose keys is clean. UserDefaults is appropriate here.

---

## 7. Timeout Logic

**The timeout is from open-time, not from last-interaction.**

`scheduleFullControlTimeout()` creates a Task that sleeps for `HearthAnimation.fullControlTimeout` (45 seconds) from when `openFullControl()` was called. Every tap on a room tile, every scroll in a drill-in, every thermostat adjustment — none of these reset the clock. The only things that touch the timeout are `openQuickControl()`, `openFullControl()`, and `dismiss()`, all of which call `cancelTimeouts()` first.

**The concrete failure mode:** User opens full control, browses through drill-ins for 40 seconds, taps a tile at 44 seconds — the panel dismisses at 45 seconds, mid-interaction, before the tap resolves. The drill-in sheet is still open. The `.onChange(of: model.panelState)` clears `drillInRoom` with `withAnimation` but the effect is jarring.

**How hard to fix:** Medium. The cleanest approach is a `lastInteractionDate: Date` property in AppModel, updated by a `func recordInteraction()` call, with the timeout loop checking elapsed time rather than just sleeping:

```swift
func scheduleFullControlTimeout() {
    fullControlTask = Task {
        repeat {
            try? await Task.sleep(for: .seconds(5))
            guard !Task.isCancelled else { return }
        } while Date.now.timeIntervalSince(lastInteractionDate) < HearthAnimation.fullControlTimeout
        dismiss()
    }
}
```

The `recordInteraction()` call needs to be wired into every interactive element in full control, which is the tedious part. A global `.gesture` overlay on `FullControlContent` might catch most cases without per-element wiring.

---

## 8. Test Coverage Gaps

Zero tests exist. Nothing is tested. Priority order if tests are ever added:

**Critical — business logic:**
- `hvacArrowConfig(for:)` — the OFF/ECO/heat/cool branching is complex and visual correctness depends on it being right
- `ClimateState.displayTargetTemp` — especially the ECO boundary selection logic
- `ClimateState.ecoActiveSide` — midpoint calculation
- `ShuffleQueue` — advance/back/antiRepeat window; edge cases (empty pool, pool smaller than antiRepeatWindow)
- `PollingHAClient.parseClimate()` — hvac_mode/preset_mode/hvac_action → ClimateMode/ClimateActivity mapping
- `PollingHAClient.parseChildren()` — recursive depth, cycle guard
- `WeatherService.precipForecast()` — 24h window logic, timezone handling

**Important — persistence:**
- `HAConfig.load()` / `HAConfig.save()` round-trip
- `PanelPreferences.load()` migration from legacy keys
- `PanelPreferences.load()` when UserDefaults is empty

**Nice to have — state machine:**
- Panel state transitions (ambient → quick → full → ambient flow)
- Timeout scheduling and cancellation
- `notifyWriteDisabled()` toast sequencing

---

## 9. Architectural Decisions I'd Reverse

**1. HA client returns a full HomeSnapshot on every poll.**

The `fetchHome() -> HomeSnapshot` protocol returns the entire home state every 10 seconds. The receiver (`applySnapshot`) overwrites everything. This means every poll triggers SwiftUI diffing across all rooms, all children, climate, house status — even when nothing changed. A push-based or delta-based model (HA's WebSocket API delivers individual entity state changes) would be more efficient and let animations be scoped to what actually changed. The current model was the right starting point but will show cracks when we have dozens of entities.

**2. `ClimateMode.allCases` drives pill rendering directly.**

The mode pills use `ForEach(ClimateMode.allCases, id: \.self)`. ECO is now a case, which means it appears in the ForEach like the others — but ECO's semantics are completely different (it's a preset, not an hvac_mode; its tap behavior would call a different HA service). Using `CaseIterable` iteration hides this distinction. A `ClimatePill: Identifiable` type with explicit `displayName`, `color`, `haServiceCall` properties would make the difference visible and testable.

**3. `AppModel` as monolith.**

AppModel is 300 lines doing: photo management, HA polling, weather polling, panel state machine, write toast, preferences, connection state. The right decomposition is probably: `PanelStateController` (state machine + timeouts), `PhotoController` (image rotation + swipe), `HAController` (polling + connection state), and `AppModel` as a thin coordinator that owns instances of each. This would make unit testing any subsystem possible.

**4. Ambient + engaged status bar as one view with opacity crossfade.**

Both layouts are always in the hierarchy. The ambient layout (with its large 68pt fonts and icon glows) is rendering even when quick/full control is showing. The engaged layout renders even in ambient. At the panel's always-on duty cycle this is probably fine, but it's architecturally wrong: two full layout trees for a component that's conceptually one. An enum-driven single layout would be leaner.

**5. The photo `transition` specifying its own animation.**

`.transition(.opacity.animation(...))` embeds animation into the transition definition. This makes the animation impossible to suppress from the call site (the `disablesAnimations` transaction issue described in #5 above). Animation should live in the call site, not in the view definition. The view should only declare the transition shape.

---

## 10. "I Knew This Wasn't Right But Shipped It"

**`hvacArrowKey` string concatenation for color identity:**

```swift
func hvacArrowKey(_ climate: ClimateState) -> String {
    guard let cfg = hvacArrowConfig(for: climate) else { return "off" }
    return "\(cfg.style.rawValue)-\(cfg.color == .hearthCool ? "cool" : "amber")"
}
```

Color equality comparison via `==` works on SwiftUI `Color` but returns true only for identical Color instances. If `hearthCool` is ever defined via an asset catalog (which looks up at render time), `==` comparison may not work as expected. And if a third color ever enters the arrow vocabulary (ECO green?), this silently produces wrong keys. The right type is a `HVACArrowColor: String { case amber, cool }` enum that the arrow config returns directly.

**`LightDrillInView` optional properties over an enum:**

Described in section 1. I knew it was wrong when I wrote it and chose speed. The three-optional pattern with computed resolvers is fragile.

**The double ZStack in `closeUpContent` for weather centering:**

The weather block uses `frame(maxWidth: .infinity)` as a full-width background layer in a ZStack, with the HStack sitting on top. This is a spatial hack — the weather isn't *centered* by layout, it's centered because the full-width frame makes it the ZStack's center. If any of the surrounding content gets wide enough to overlap, the weather reads as off-center. The correct approach is equal-width columns or `GeometryReader`.

**All polling loops are unmanaged:**

`startPhotos()`, `startWeather()`, `startHA()` all spawn Tasks that run forever. I noted this every time and shipped it. They're safe in practice (the model lives for app lifetime, the Tasks die with it) but wrong in principle. And the re-entry bug in `startHA()` (described in section 2) is a real consequence of this.

**Version string in SettingsView:**

`LabeledContent("Version", value: "0.3.0-phase0c")` — already stale at v0.3.8. I put it there as a placeholder and never wired it to `BuildInfo` or `Bundle.main`. It will always be wrong.

**`MockHAClient.fetchHome()` is `async` but not `throws`:**

The protocol requires `async throws`. `MockHAClient` conforms with just `async` (Swift allows this). It works but any future reader might add a try/catch expecting it could fail. A comment noting the intentional omission would help.

---

## Summary Priority

If I had one sprint to spend on health:

1. **Fix the stacking HA polling loop** (section 2) — it's a real concurrent bug that worsens with use
2. **Split `MainPanelView.swift`** into ~5 files (section 1) — necessary before any more features land
3. **Move token to Keychain** (section 6) — it's a credential
4. **Fix the photo swipe animation** (section 5) — users notice this
5. **Unify opacity/color constants** (section 3) — reduces future drift
6. **Interaction-based timeout reset** (section 7) — user experience correctness
7. **Refactor `LightDrillInView` source enum** (section 10) — before recursion gets deeper
