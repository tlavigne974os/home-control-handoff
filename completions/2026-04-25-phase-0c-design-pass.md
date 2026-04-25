# Phase 0c Completion Report — Design Pass + Lighting Drill-In

**Date:** 2026-04-25  
**Tag:** v0.3.5  
**Deployed to:** Todd's iPad Pro 12.9" + iPad Air 13" M2  

---

## What was built

### Room grouping (finalized)
Collapsed from 6 rooms to 4 master group tiles matching real Home Assistant light groups:
- **Upstairs** (`light.upstairs_master`)
- **Downstairs** (`light.downstairs_master`)
- **Todd** (`light.todd_master`)
- **Stairs & Halls** (`light.stairs_and_hall_master`)

Sub-room drill-down is a future phase — not in scope yet.

### Lighting drill-in (Phase 0c.5)
Tapping a room tile in Full Control opens a full-screen sheet showing per-zone child lights. Children are discovered dynamically from HA's `attributes.entity_id` — no hardcoding. Each row shows a brightness bar, unavailability state ("Unavailable" in hearthAlert red), and a nested group badge for rooms-within-rooms.

### Design pass (Phase 0c.6) — 8 refinements
1. Status bar door/garage icons enlarged to 48pt, wrapped in tappable buttons
2. Full-control door/garage icons scaled up (40pt glyph, 72pt glow)
3. Full-control mic icon 48pt (was 36pt)
4. Trinity + ModePill hidden in full control (cleaner full-screen layout)
5. Thermostat mode pills moved LEFT, −/+ on RIGHT in both quick and full control
6. Mic removed from quick actions row
7. Bottom action row redesigned: two-line tiles (verb+noun / status sub-label)
8. Home mode tile: gold border + "CURRENT MODE" sub-label for active mode

### Photo positioning
Photo bottom edge in ambient state now sits halfway down the status bar (`ambientStatusBarOverlap = 0.5`), creating a subtle layering between photo and status bar. Tunable constant.

### WeatherKit
Confirmed live in provisioning profile and returning data. Status bar shows `current → high` in hearthGold (e.g. `68° → 71°`).

---

## Screenshots

See `screenshots/2026-04-25/` in this repo:
- `ambient.png` — ambient state (dark background, status bar, no album selected)
- `quick-control.png` — quick control panel (~40% height, photo dimmed)
- `full-control.png` — full control (climate card + lighting grid + action bar)

---

## Design decisions CC made autonomously

- Weather format: `current → high`, no icon, hearthGold color
- Quick control climate row: mode pills left, −/+ right
- Full control thermostat: Fraunces italic for inline target temperature value
- Action tiles: two-line (verb+noun / status), current mode tile gets gold border
- Photo overlap tunable constant (0.5 = halfway into status bar)

---

## Outstanding design questions for DC

1. **Lighting drill-in sheet** — currently a plain dark list with brightness bars. Needs a design pass. Sheet vs. slide-in? How to show nested groups? Unavailable zone treatment?
2. **Nav icons (full control bottom bar)** — 5 placeholder icons (cart, history, home, broadcast, settings). What are the real destinations and iconography?
3. **Weather in ambient** — brief calls for a condition icon. Where does it live? Should ambient show anything weather-related beyond the status bar?
4. **Voice/mic button** — currently in the status bar right cluster. Brief implies something more prominent. Intentional placement or placeholder?
5. **Thermostat "eye-level" label** — HERE temp is labeled "eye-level" (sensor location). Is this copy intentional?
6. **Phase 0d interaction model** — all taps show a "writes coming soon" toast. Before wiring HA writes: tap-to-toggle for lights? Hold for brightness? Confirm/cancel for door unlock?
