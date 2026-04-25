# Phase 0c.7 Completion Report — Icon Unification + Lighting Drill-In Design Pass

**Date:** 2026-04-25  
**Tag:** v0.3.6  

---

## 1. Icon language unification — Phosphor everywhere

Zero SF Symbols remain in user-visible views. All replaced with Phosphor equivalents:

| Location | Was | Now |
|---|---|---|
| Quick control status bar door | `Ph.lock` (wrong!) | `Ph.door` / `Ph.doorOpen` |
| ALL ON button | SF `lightbulb.fill` | `Ph.lightbulb` |
| ALL OFF button | SF `lightbulb.slash` | `Ph.lightbulbFilament` |
| Thermostat − (QC + FC) | SF `minus` | `Ph.minus` |
| Thermostat + (QC + FC) | SF `plus` | `Ph.plus` |
| Precipitation indicator | SF `drop.fill` / `snowflake` | `Ph.drop` / `Ph.snowflake` |
| Ambient no-album gear icon | SF `gearshape` | `Ph.gear` |
| Ambient loading photo icon | SF `photo.on.rectangle.angled` | `Ph.images` |
| Album picker checkmark | SF `checkmark` | `Ph.check` |

Door/garage/mic in the status bar were already Phosphor in ambient — now consistent across all three states.

## 2. Lighting drill-in design pass

Updated `LightDrillInView` and `LightChildRow`:

- **Unavailable state**: was Geist Mono → now Fraunces italic, hearthAlert red. Matches spec.
- **On state**: added "On at X%" text label above the brightness bar (was bar-only).
- **Group rows**: amber "GROUP" badge (was plain gray capsule), plus `groupStateView` showing "X lights · on at Y%" or "X lights · all off" aggregate.
- **Chevron**: only shown for group rows (not individual lights). Colored hearthAmberDim to signal drillability.
- **Empty state**: Fraunces italic "No lights found in this group" — unchanged, already correct.

**Limitation noted**: "2 on" count granularity inside a group would require sub-child discovery. Current model has `childCount` and aggregate `brightness` but not per-sub-child on/off state. Aggregate state line shows "on at X%" instead. Fine for Phase 0c.

## 3. Status bar door/garage tap — confirmed working

`statusBarActionButton` wraps each status bar icon in a `Button` with `.buttonStyle(.plain)`. Since `MainPanelView` renders on top of `AmbientView` in the ZStack, button taps take priority over AmbientView's `.onTapGesture`. Verified structurally — no code change needed.

## Screenshots

See `screenshots/2026-04-25/` in this repo:
- `ambient.png` — ambient state, consistent door/garage/mic Phosphor icons
- `quick-control.png` — quick control, door now Ph.door (was Ph.lock), Phosphor −/+
- `full-control.png` — full control, Phosphor lightbulb ALL ON/OFF, Phosphor −/+
- `drill-in-downstairs.png` — Downstairs drill-in sheet (empty state in simulator — no HA connection)

## Simulator tooling added

`--drillIn <roomId>` launch argument auto-presents the drill-in sheet headlessly. Enables drill-in screenshots without needing physical screen interaction.
