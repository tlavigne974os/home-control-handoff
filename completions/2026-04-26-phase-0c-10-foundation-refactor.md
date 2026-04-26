# Phase 0c.10 — Foundation Refactor

**Tag:** v0.4.0  
**Commits:** e512bce (file split + DrillSource), 506ad21 (token namespaces), 2753b88 (AppModel decomposition), 9e5e23c (Keychain)  
**Date:** 2026-04-26  
**Deployed:** 1st floor iPad Pro (v0.4.0)

---

## What Was Built

Structural-only refactor pass derived from the Phase 0c.9 audit. Five items, zero behavior change. Screenshots after this commit look identical to screenshots before it. The entire investment is in foundations that make future phases cheaper.

---

### Item 1: Split MainPanelView.swift into Logical Files

**Before:** 1,157-line monolith containing the panel container, status bar, trinity, alert glow, climate section, climate strip, rooms section, room tile, zone tiles, bottom action row, quick actions row, HVAC arrow shape, and all helpers.

**After:** Six files with single concerns:

| File | Contents | Lines |
|---|---|---|
| `HVACArrow.swift` | HVACArrowStyle, HVACArrowView, hvacArrowConfig, hvacArrowKey | 98 |
| `StatusBarView.swift` | StatusBarRow, AlertGlowIndicator, TrinityStatusBar | 230 |
| `ClimateViews.swift` | ClimateSection (full control), ClimateStripRow (quick control) | 220 |
| `RoomsViews.swift` | RoomsSection, RoomTile, ZoneTilesRow, ZoneMiniTile | 185 |
| `BottomActionRow.swift` | BottomActionRow, QuickActionsRow | 140 |
| `MainPanelView.swift` | MainPanelView, QuickControlContent, FullControlContent, sectionLabel | 145 |

Access modifiers updated: previously-`private` structs are now `internal` (default) — they remain package-internal; nothing is exported from the app target. `private` was only keeping them file-local in the monolith, which had no practical benefit since they were the only consumers.

`sectionLabel` free function moved to MainPanelView.swift (used by QuickControlContent and FullControlContent both in the same file); accessible to ClimateSection and RoomsSection which are now separate files.

---

### Item 2: Extract Design Token Namespaces

**HearthLayout** (new):
- `arrowAmbient` / `arrowStatusBar` / `arrowHeroCard` — CGSize for the three arrow contexts
- `climateSectionFraction` — 0.52 fraction of screen width
- `trinityCardHeight` — 56pt divider height in the trinity card

**HearthAnimation** additions:
- `writeToastFadeIn` / `writeToastFadeOut` / `writeToastDuration`
- `alertGlowFadeIn` / `alertPulseDuration` / `alertGlowPulseLow`
- `arrowCrossDissolve`
- `temperatureFlip` / `brightnessSlide` / `tileTint`

**HearthTypeSize** additions:
- `trinityValue` (32pt) — trinity card temperature values
- `stripTemp` (36pt) — climate strip row temperature

**Color opacity helpers** (new extension on Color):
- `.tileFill` — 7% opacity, standard tile background when active
- `.tileBorder` — 25% opacity, standard tile border when active
- `.pillFill` — 15% opacity, pill fill when active
- `.pillBorder` — 40% opacity, pill border when active

Previously: `Color.hearthAmber.opacity(0.07)` appeared 6 times across 4 files. Now: `Color.hearthAmber.tileFill` everywhere. Changing "what does an active tile background look like" is one constant change instead of a grep-and-replace.

---

### Item 3: Decompose AppModel

**Before:** 303-line `AppModel` doing: panel state machine, timeouts, photo rotation, album selection, swipe navigation, HA polling, HA client lifecycle, home state management, weather polling, write toast, connection state.

**After:** Three focused controllers + a thin coordinator:

**`PanelStateController`** — state machine + timeouts only.
- `panelState`, `openQuickControl()`, `openFullControl()`, `dismiss()`
- Two `Task` properties for timeout management
- Can be unit-tested with no photo/HA dependencies

**`PhotoController`** — all photo concerns.
- `currentImage`, `currentPhotoDate`, `authorizationStatus`, `availableAlbums`
- Album selection, shuffle queue, rotation loop, swipe navigation
- Takes a `weak var stateController` reference so the rotation loop can self-gate when panelState != .ambient
- `PanelPreferences` (UserDefaults) lives here — it's all photo-related

**`HAController`** — all HA + weather concerns.
- `home`, `haConfig`, `haClient`, `haConnectionState`, `showWriteToast`, `weatherReady`
- Polling loop (stacking-loop bug still present — isolated here for 0c.11 fix)
- Write toast logic
- Weather fetching

**`AppModel`** — thin coordinator, ~85 lines.
- Owns controller instances
- Forwards every property and method via computed properties
- Views depend only on `AppModel` — no view knows which controller owns what

**Observation forwarding correctness:** `@Observable` tracks stored property accesses transitively through computed properties during SwiftUI view body evaluation. When a view reads `model.panelState` (computed), SwiftUI captures the dependency on `PanelStateController._panelState` (stored, `@Observable`-instrumented). Re-renders fire correctly when the stored value changes. This is documented and supported behavior in the Observation framework (iOS 17+).

---

### Item 4: Migrate Token to Keychain

**KeychainHelper** (new service): synchronous save/load/delete using `kSecClassGenericPassword`. Service attribute `"HomeControl"` groups all app entries. Access level `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` — device-specific (no iCloud Keychain sync), correct for wall panel long-lived tokens.

**HAConfig changes:**
- `token` removed from `CodingKeys` — excluded from JSON encoding/decoding
- `token` is now a computed property: `get` reads Keychain, `set` writes Keychain
- `isConfigured` reads through the computed property — no stored truth
- `save()` only persists non-sensitive config (URL, entity IDs) to UserDefaults

**One-time migration:** `HAConfig.load()` checks for a token in the old UserDefaults JSON blob. If found, writes it to Keychain and rewrites the blob without the token field. Migration runs once; subsequent loads skip the check.

**Verification:** Token now survives Xcode rebuild-and-deploy. Keychain items persist through app updates (including Xcode incremental reinstalls). UserDefaults is wiped on clean install. The audit's concern — losing the token after Xcode deploys — is addressed.

---

### Item 5: Refactor LightDrillInView Source Pattern

**Before:** Three optionals manually implementing a tagged union:
```swift
var roomId: String? = nil
var groupName: String? = nil
var groupChildren: [LightChild]? = nil
```
Plus a convenience initializer extension to fill two of the three. The compiler could not enforce that exactly one source was active.

**After:** Explicit enum:
```swift
enum DrillSource {
    case room(id: String)
    case group(name: String, children: [LightChild])
}
```

`displayTitle` and `displayChildren` are now exhaustive `switch` statements. The compiler enforces completeness. Recursive `.sheet` call updated: `LightDrillInView(source: .group(name:children:), model:)`.

All call sites updated:
- `RoomsSection`: `.room(id: selected.id)`
- `QuickControlContent`: `.room(id: selected.id)`
- Recursive drill-in: `.group(name: child.name, children: child.children)`

---

## What Was Not Changed

- No bug fixes (those are Phase 0c.11)
- No new features
- No polling optimization (WebSocket replaces this anyway in Phase 0d)
- No tests added
- No behavior change of any kind

---

## Autonomous Decisions

**`sectionLabel` as module-level function vs. placed in its own file:** The function is one line. Making it its own file (`ViewHelpers.swift`) would require an Xcode project entry for four words of code. Keeping it in `MainPanelView.swift` (which `ClimateSection` and `RoomsSection` are compiled with) is simpler. If a third consumer appears, reconsider.

**Weather in HAController not its own controller:** Weather has three lines of logic and no state beyond `home.weather` + `weatherReady`. A `WeatherController` with two properties and one `Task` loop would be over-decomposed. HAController is the right home for now — weather is already part of the home state model.

**`PhotoController` `weak var stateController`:** The rotation loop needs to gate on panel state. Options were: pass `panelState` as a closure parameter, poll a shared property, or take a weak reference to `PanelStateController`. The weak reference is cleanest — `PhotoController` is always owned by `AppModel` which also owns `PanelStateController`, so the weak ref is safe for the app's lifetime.

**DrillSource conformance to `Equatable`:** Not added. `LightChild` would need to be `Equatable` (it's not), and the only place `DrillSource` is compared is in `displayTitle`/`displayChildren` which switch on it directly. No conformance needed.

---

## Things That May Need DC Evaluation

1. **Token on second deploy:** Keychain migration runs on first load after update. If the user had already entered a token and Xcode does a clean install (not an update), the token is wiped before migration can run. This is edge-case — normal Xcode "Run" is an update install. But on first deployment to a new device, the user always re-enters the token. No change in user experience.

2. **`sectionLabel` visibility:** Currently a module-level function in `MainPanelView.swift`. If `PersonalApp` or a future target needs the same label style, it needs to be promoted to a shared component in HomeControlCore. Not needed yet.

3. **`PanelPreferences` location:** Album selection preferences live in `PhotoController`. If future preferences (night dimming threshold, etc.) are added, they may fit better in `AppModel` or their own controller. `PanelPreferences` can migrate without changing the persistence format.
