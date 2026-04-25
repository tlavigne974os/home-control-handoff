# Phase 0c.7 — Quick Control Rework + Status Bar Refinements + Bug Fixes

**Tag:** v0.3.7  
**Commits:** f294249 (panel), 17b21c0 (weather/deploy)  
**Date:** 2026-04-25

---

## What Was Built

### Part 1+2: Status Bar Parity + State Pill Relocation

The engaged status bar (quick control and full control) is now identical — same trinity, same weather, same icons, same layout. The state pill (IDLE/HEATING/COOLING) was removed from the status bar entirely. It now lives:

- **Quick control**: inline in the climate strip row, between "toward X°" and the mode pills
- **Full control**: the climate hero sub-line ("Heating toward X°") was made activity-aware — it now correctly says "Heating toward", "Cooling toward", "Idle · set to", or "Off · set to" depending on `climate.activity`

The old `ModePillStatusBar` view was removed. A new `InlineActivityPill` was added for the compact strip version — same pulsing dot behavior, sized for the narrower row context.

### Part 3: Status Bar Refinements

1. **Arrow direction reversed**: Ambient two-temp now shows `systemTemp → targetTemp` (73° → 68°). Trinity status bar reordered to `SYSTEM → TARGET · HERE`. Full control trinity card also reordered. Reads as: "the system is currently at X°, moving toward Y°".

2. **Weather conditions in engaged bar**: Already present — confirmed correct. The centered `VStack` shows temp + condition string + precip if any.

3. **Weather block centered**: Implemented via `ZStack` — the weather `VStack` expands to full bar width (`frame(maxWidth: .infinity)`) as the background layer, truly centering it. The left (time/date) and right (trinity+icons+mic) sit in an `HStack` on top.

4. **Two-line SATURDAY/AFTERNOON**: `periodLabel` split into `dayLabel` ("Saturday") and `timePeriodLabel` ("Afternoon"), each rendered as separate `Text` in the engaged bar's left `VStack`. Ambient keeps the combined single-line treatment.

### Part 4: Quick Control — Climate Strip

Replaced `ThermostatRow` with `ClimateStripRow`. Single-line layout left to right:
```
CLIMATE  73°  toward 68°  ·  [IDLE]  [HEAT] [COOL] [OFF]  [−] [+]
```
The `−/+` buttons sit immediately after the mode pills (no longer far-right and disconnected). `Spacer(minLength: 0)` at the end absorbs remaining space.

### Part 5: Quick Control — Zone Tiles

Replaced the single `LightsRow` slider with `ZoneTilesRow` — 4 mini tiles driven by `model.home.rooms`. Each tile shows zone name, "On at X%" or "Off", and a brightness bar when on. Tapping any tile opens `LightDrillInView` for that zone (same drill-in as full control). The `QuickControlContent` owns the `drillInRoom` state and the `.sheet` modifier.

### Part 6: Trinity Deduplication

With Parts 1+2 applied, full control's status bar now shows the trinity summary (SYSTEM→TARGET·HERE), and the climate hero card shows the expanded trinity with sub-labels and activity-aware text. State pill is only in the hero. This is intentionally layered: status bar = glance, hero = detail.

### Part 7: Drill-In Bugs

**Bug 1 — Group chevrons removed**: `LightChildRow` no longer shows the `Ph.caretRight` chevron on group rows. The GROUP badge remains (honest label), but the chevron that implied tappable drill-down is gone. Group rows still fire `notifyWriteDisabled()` on tap (which is honest — toggling a group is a write). Proper recursive drill-down deferred to a future phase when `LightChild` has a `children` array.

**Bug 2 — Sheet dismissal on ambient transition**: Three places were fixed:
- `RoomsSection` (full control): `.onChange(of: model.panelState)` clears `drillInRoom` when `.ambient`
- `BottomActionRow` (full control): `.onChange` clears `showSettings` when `.ambient`
- `QuickControlContent` (new): `.onChange` clears its own `drillInRoom` when `.ambient`

---

## Autonomous Design Decisions

**ZStack centering for weather**: Used `ZStack` with weather as the full-width background layer and time/right-side in an `HStack` overlay. An alternative 3-column approach would pixel-perfect center the weather but would make the right side (trinity + 2 icons + mic) too narrow. The ZStack approach gives true visual centering without constraining the right side.

**Trinity column order in status bar**: Changed from `TARGET → SYSTEM · HERE` to `SYSTEM → TARGET · HERE`. The old order put the goal first (aspirational read), the new order puts current state first (diagnostic read). This matches the arrow semantics: "system is at X, heading to Y." Spec was clear on the direction; I chose reordering over flipping the arrow to `←` because `SYSTEM → TARGET` is more scannable.

**`activityLabel` in ClimateSection**: The "Heating toward" text was hardcoded. I made it dynamic: `.heating` → "Heating toward", `.cooling` → "Cooling toward", `.idle` → "Idle · set to", `.off` → "Off · set to". The `·` separator before the sub-label in idle/off states avoids the awkward "Idle toward 68°" phrasing.

**Group badge kept, chevron removed**: The GROUP capsule badge still appears on group rows — it's accurate metadata (this IS a HA group, not a single bulb). Only the chevron is removed, since it implied navigability that doesn't exist yet. This is a clean distinction.

**`ZoneMiniTile` uses `minimumScaleFactor(0.7)`**: "Stairs & Halls" is long. Without scale factor it would clip on some zone widths. This keeps all 4 tiles equal width and the name readable.

---

## Things That May Need DC Evaluation

1. **Engaged status bar density**: The right side now has trinity + 2 icon buttons + mic. On the iPad's wide landscape bar this should breathe fine, but the trinity (3 columns) + icon glow areas + mic could feel crowded on narrower devices or if we ever add more right-side elements. Worth a real-device look.

2. **Climate strip row length**: The full strip (`CLIMATE | 73° | toward 68° | · | IDLE | HEAT COOL OFF | − +`) is dense. It fits the math but may feel tight at the actual font sizes on device. The `Spacer(minLength: 0)` absorbs slack — if it looks crowded, dropping "CLIMATE" label or making the mode pills smaller are the obvious relief valves.

3. **Two-line day/period in engaged bar**: The left block is now 3 lines (time + day + period). This is taller content than the single-line it replaced. At `statusClock` = 44pt + 2×15pt Geist Mono labels, it fits the 130pt status bar height but the visual weight has shifted left. DC should check whether the overall bar still feels balanced.

4. **Full control trinity duplication**: The trinity is now in both the status bar (summary, compact) and the climate hero card (expanded, with sub-labels). The spec says to try it and see — flagging this for evaluation. If it feels redundant, the clean resolution is to drop the trinity from the full-control status bar only, showing just the weather centered and the icons/mic right.

5. **Group drill-down (deferred)**: Groups in the drill-in sheet now have no chevron. If DC's design work for Phase 0e includes a second-level drill-in, implementing it will require adding `children: [LightChild]` to `LightChild` in HomeModels and populating it from `PollingHAClient`. The data model change is the blocker, not the view.

---

## Outstanding Questions for DC

- Should the ambient two-temp ever show the state pill? Currently ambient shows the raw numbers with a colored arrow. Full context (HEATING/IDLE) is only in engaged.
- Is the GROUP badge still useful in the drill-in without the chevron, or does it just add noise? Could remove it too.
- The weather condition string (e.g., "PARTLY CLOUDY") in the centered block — should it truncate or scale if the condition string is long ("THUNDERSTORMS WITH HEAVY RAIN")?
- For the zone mini-tiles in quick control — should "Stairs & Halls" be renamed to something shorter (e.g., "Halls") to avoid scaling? That's a data change in HardcodedData and PollingHAClient.
