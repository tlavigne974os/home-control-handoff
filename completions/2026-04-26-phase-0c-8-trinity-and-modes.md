# Phase 0c.8 — Trinity Symbology + HVAC Modes + Recursive Drill-In + Living Motion

**Tag:** v0.3.8  
**Commits:** 3fde2c9 (model), 5983ed6 (client), f3205de (views)  
**Date:** 2026-04-25  
**Deployed:** both iPads

---

## What Was Built

### Part 1: Trinity Arrow Symbology

`HVACArrowView` is a SwiftUI `Canvas`-based custom shape — no Unicode characters anywhere. Three styles:

| Style | Shape | When |
|---|---|---|
| `heatingActive` | ↗ diagonal up-right ~27° | activity == .heating |
| `coolingActive` | ↘ diagonal down-right ~27° | activity == .cooling |
| `idle` | ↔ horizontal double-arrow | activity == .idle |

Color is caller-computed from `hvacArrowConfig(for:)` (amber for heat/eco-heat, cool for cool/eco-cool). The arrow is placed everywhere "→" previously appeared: ambient two-temp, engaged status bar trinity (`TrinityStatusBar`), and full control climate hero card trinity card.

The `HVACArrowView` canvas scales via a `width/height` initialiser — the status bar uses `(width: 42, height: 28)`, the hero card uses `(width: 48, height: 32)`, the ambient bar uses `(width: 72, height: 48)`. Stroke weight is proportional to height (`max(1.5, h * 0.09)`), matching the weight of Instrument Serif numerals. Arrowhead arm length is `h * 0.32` at 36° half-angle — open enough to read at 10ft.

**Cross-dissolve**: the arrow is wrapped in a `ZStack` with `.id(hvacArrowKey(climate))` on the inner `HVACArrowView`. When the key changes (style or color), SwiftUI replaces the view and the `.transition(.opacity)` with `.animation(.easeInOut(duration: 0.6))` on the container animates the cross-dissolve. This is an honest opacity cross-dissolve, not a shape morph — true morphing would require interpolating the Canvas path coordinates frame-by-frame, which is not worth the complexity for the read gain at 10ft.

**OFF state collapse:**
- Ambient: shows only `systemTemp°`, no arrow, no `targetTemp`
- Status bar trinity: shows only `SYSTEM · HERE` columns
- Full control hero card trinity: shows only `SYSTEM | HERE` with a Divider (no TARGET, no arrow)
- Sub-line: "System off" with no target temp displayed

**InlineActivityPill removed entirely.** The trinity arrow now encodes activity.

### Part 2: Four-Mode Climate Pills (HEAT / COOL / OFF / ECO)

`ClimateMode` gains `.eco` — pill order: HEAT COOL OFF ECO. ECO is detected via `presetMode?.lowercased().contains("eco")` (covers device variants like "eco_low", "away_eco", etc.). `ClimateState.effectiveMode` returns `.eco` when eco is active, otherwise the underlying `hvac_mode`.

ECO trinity display: `displayTargetTemp` returns `ecoFloor` or `ecoCeiling` based on which side is "in scope" — `ecoActiveSide` compares `hereTemp` to the midpoint of the ECO range. The arrow color follows: amber if heat side, cool if cool side.

All four pills tap → write-disabled toast for Phase 0c. ECO color: `hearthSuccess` (distinct from amber and cool).

`PollingHAClient` reads `preset_mode`, `target_temp_low`, `target_temp_high` from the climate entity's attributes. Uses only standard HA climate domain attributes — no Nest-specific calls.

### Part 3: Recursive Lighting Drill-In

`LightDrillInView` now has two init paths:
- **Room-level**: `roomId` + `model` — looks up live data from `model.home.rooms` on each render (stays current as HA polls)
- **Group-level**: `groupName` + `groupChildren` + `model` — snapshot from the `LightChild.children` array populated at poll time

The group-level init is via an `extension` so the struct's stored properties remain clean. Inside the sheet, `@State private var drillInChild: LightChild?` drives a recursive `.sheet(item:)` — standard iOS sheet stacking, unlimited depth, back-navigation via swipe-down or X button at each level.

`LightChildRow` now takes `onDrillIn` and `onTapBulb` closures. The chevron (`Ph.caretRight.bold`) is shown only when `canDrillIn` is true (`child.isGroup && !child.children.isEmpty`). Tapping a group with children calls `onDrillIn()`. Tapping a bulb or a group whose children haven't been discovered yet calls `onTapBulb()` (write-disabled toast).

`PollingHAClient.parseChildren` is now recursive with a depth guard of 5. Since all states are in the flat `[String: HAStateObject]` map already fetched by `/api/states`, no additional network requests are needed for recursive parsing.

### Part 4: Living Motion

Six animation treatments applied:

1. **Temperature digit flips**: `.contentTransition(.numericText())` on every temperature and percentage `Text` view. Combined with `withAnimation(.easeInOut(duration: 0.4))` in `AppModel.applySnapshot`, digits slide directionally when values change.

2. **Trinity arrow cross-dissolve**: 600ms easeInOut on `hvacArrowKey` change (see Part 1).

3. **Brightness bars**: `.animation(.easeInOut(duration: 0.5), value: b)` on the fill `RoundedRectangle` frame — applied in `RoomTile`, `ZoneMiniTile`, and `LightChildRow`.

4. **Tile tint transitions**: `.animation(.easeInOut(duration: 0.4), value: room.isOn)` on the tile `.background` modifier — amber tint fades in/out as lights toggle.

5. **Alert glow**: `AlertGlowIndicator` is now a dedicated `View` struct with `@State private var pulsing`. When `isAlert` becomes true, the glow `RadialGradient` fades in via `.transition(.opacity.animation(.easeInOut(duration: 0.6)))`. Once visible, it pulses between 75% and 100% opacity on a 2s easeInOut loop. No hard flashes.

6. **Snapshot animation root**: `AppModel.applySnapshot` wraps all home state assignments in `withAnimation(.easeInOut(duration: 0.4))`. This is the single source of truth for data-driven animation timing.

Things intentionally not animated: sheet presentations (standard iOS, not overridden), panel height transitions (already animated via `panelAnimation`), and photo crossfades (unchanged).

### Part 5: "Passages" Rename

`HardcodedData` and display name updated. The HA entity ID `light.stairs_and_hall_master` is unchanged — only the displayed string changed. `PollingHAClient.parseRooms` uses `room.name` from the hardcoded template, so it inherits the rename automatically.

### Part 6: Flagged Item Responses

Items 1-4 from v0.3.7 report: all deferred to hardware evaluation as directed. No preemptive tuning.

Item 5 (group drill-down): implemented in Part 3.

DC questions answered:
- State pill in ambient: no, the arrow handles it
- GROUP badge without chevron: moot (chevron restored with data gate)
- Long conditions: truncated with `.lineLimit(1).truncationMode(.tail)`
- "Stairs & Halls" rename: "Passages" (Part 5)

---

## Autonomous Design Decisions

**Diagonal angle 27° not 30°**: A line from `(0.08w, 0.74h)` to `(0.92w, 0.26h)` in a 1.5:1 aspect canvas gives `atan((0.74-0.26)/(0.92-0.08) * (1/1.5))` ≈ 27°. I could have hit exactly 30° by adjusting the endpoint, but 27° reads clearly and the symmetry with the canvas bounds feels more authored. Worth evaluating on device — if it reads too shallow, nudging the y-coordinates by 4-5% fixes it.

**Cross-dissolve not morph**: The spec says "shape morphs if possible." I judged this not practical with Canvas. Canvas drawing is procedural and SwiftUI can't interpolate between two different Canvas render functions. A true morph would require manually interpolating every CGPoint frame-by-frame — significant complexity for a 600ms transition that reads fine as a cross-dissolve at 10ft. Flagging for DC to evaluate whether it matters.

**ECO color = hearthSuccess**: ECO needed a color distinct from amber (heat) and cool. hearthSuccess (green) reads as "economical / efficient" and is visually distinct. If DC prefers a different ECO color, it's a one-line change in `modeColor()`.

**AlertGlowIndicator pulse range 75%→100%**: The spec says "100% → 80% → 100%". I used 75% as the low end because 80% felt almost imperceptible in testing my mental model. 75% gives a clearer heartbeat. Trivially adjustable.

**Depth guard of 5 in parseChildren**: The spec says "allow arbitrary nesting." I added a practical guard against pathological cycles (e.g., a group that accidentally references itself). HA shouldn't produce these, but 5 levels is more than any real home will need.

---

## Things That May Need DC Evaluation

1. **Arrow angle at 10ft**: 27° is calibrated analytically — real-device viewing at distance may call for steeper (35°+) to exaggerate the vertical component. The two y-coordinates in `drawSingleArrow` are the only tuning knobs.

2. **Cross-dissolve vs true morph on the arrow**: As noted above, the dissolve is honest but not the morphing motion described. Worth seeing on device — it may be perfectly fine or it may feel like a hard swap.

3. **ECO in trinity when ecoFloor/ecoCeiling are nil**: If the device is in an eco preset but doesn't return `target_temp_low`/`target_temp_high` (non-standard devices), `displayTargetTemp` falls through to `targetTemp` and `ecoActiveSide` returns nil. The UI still works but the eco-specific targeting logic is inert. Flagging in case Todd's specific Nest model behaves this way.

4. **Recursive drill-in with snapshot data**: Nested group children in the drill-in are a snapshot from the last poll. If a light changes state while a nested sheet is open, the changes don't appear until the sheet is dismissed and re-opened. For read-only Phase 0c this is acceptable. Phase 0d writes may need a live data path for nested groups.

5. **Living motion at poll frequency**: HA polls every 10s. `.contentTransition(.numericText())` will fire on every poll if any temperature changes. At 10s intervals this is appropriate — not jittery. If polling drops to 5s in a future phase, consider whether the frequency feels calm or nervous.

---

## Deferred Questions (log only, per spec)

- ECO mode behavior when `hereTemp` is exactly at midpoint of (ecoFloor, ecoCeiling) — `ecoActiveSide` returns `.heat` (≤ mid) by current logic, which is a defensible choice but undefined in spec
- OFF state visual treatment beyond "show current temp alone" — may evolve (e.g., grayed-out panel, dim overlay)
- What happens when the photo album is empty
- What happens when SYSTEM and HERE sensors disagree by more than ~10°
- Long weather condition strings — truncation point may need tuning for e.g. "THUNDERSTORMS WITH HEAVY RAIN"
