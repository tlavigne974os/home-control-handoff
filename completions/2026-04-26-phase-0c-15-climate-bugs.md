# Phase 0c.15 — Climate Hero Bug Fixes
**Date:** 2026-04-26  
**Tag:** v0.5.3-bugfix  
**Build:** 0.5.3 / build 61  
**Status:** ✅ Complete — deployed to iPad Pro + iPad Air

---

## Bug 1 — Climate Sentence Italics ✅ Fixed

### Problem
Dynamic parts of the climate sentence (temperature value and "in the [Room]." trailing fragment) were rendering in `frauncesItalic` — italic Fraunces. The spec calls for the entire sentence to render upright Fraunces. No italics anywhere in the climate sentence.

### Fix
Replaced all `.frauncesItalic(size: HearthTypeSize.bodyMd)` with `.fraunces(size: HearthTypeSize.bodyMd)` in `subLine` and `ecoSubLine` in `ClimateViews.swift`.

All seven sentence patterns affected:
- "Currently heating toward [N]° in the [Room]."
- "Currently cooling toward [N]° in the [Room]."
- "Currently idle with the temp set to [N]° in the [Room]."
- "Currently off."
- "Currently heating toward eco floor of [N]° in the [Room]."
- "Currently cooling toward eco ceiling of [N]° in the [Room]."
- "Currently in eco mode, holding between [N]° and [N]° in the [Room]."

**Not touched:** Trinity card sub-labels (`thermostat`, `what we set`, `kitchen`) and TARGET column italic temp emphasis — those remain italic per spec.

---

## Bug 2 — Sentence/Arrow State vs hvac_action (Principle 14 Audit) ✅ No Fix Needed

### Finding: Possibility (A) — Panel faithfully reads hvac_action from HA

The panel does NOT infer active/idle from mode. The code path is correct.

In `PollingHAClient.swift`:

```swift
let hvacAction = attrs.string("hvac_action") ?? "idle"

let activity: ClimateActivity = {
    switch hvacAction {
    case "heating": return .heating
    case "cooling": return .cooling
    case "off":     return .off
    default:        return .idle
    }
}()
```

`hvac_action` is read directly from the HA entity attributes and mapped to `ClimateActivity`. Both the sentence selection (`switch climate.activity`) and the arrow rendering (`hvacArrowConfig(for: climate)`) consume `climate.activity` — they share the same derived value, derived from the same upstream source.

### What the screenshot showed

System = 69°, target = 69°, sentence = "Currently heating toward 69°", arrow = ↗.

This is **Possibility (A)** — HA was reporting `hvac_action: "heating"` at that moment. This is normal thermostat behavior: the system can be actively heating even when it reaches the target temperature during a cycle (e.g., overshoot, sensor lag, heat exchanger still warm). HA's `hvac_action` reflects the actual HVAC relay state, not a comparison of temperatures.

The panel displayed exactly what HA reported. No Principle 14 violation.

### No code change made to Bug 2.

---

## Files Changed

| File | Change |
|------|--------|
| `WallPanel/Views/ClimateViews.swift` | `frauncesItalic` → `fraunces` in all climate sentence patterns |
| `WallPanel/Info.plist` | Version 0.5.3, build 61 |

---

## Commit

Single commit covering both bugs (one is a fix, one is an audit with no change):

> `ui: climate sentence — remove italics; audit Principle 14 (hvac_action read path confirmed correct)`

---

## Next Phase

**Phase 0d.2 — Write implementation.** All 8 stubs on `HAController` throw `.notImplemented`. Tapping +/−, mode pills, lock/garage shows toast. This is the next functional phase.
