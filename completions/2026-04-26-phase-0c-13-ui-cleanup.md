# Phase 0c.13 — Status Bar Invariant + UI Consolidation
**Date:** 2026-04-26  
**Tag:** v0.5.0-ui-cleanup  
**Build:** 0.5.0 / build 50  
**Status:** ✅ Complete — deployed to iPad Air + iPad Pro

---

## Original DC Prompt

> **Phase 0c.13 — Status bar invariant + UI consolidation**
>
> Tag: v0.5.0-ui-cleanup. Substantial UI consolidation pass. Eight focused changes that work together to fix accumulated drift and lock in the status bar invariant.
>
> **PRELIMINARY: RE-READ** — DC has updated the brief. Re-read home-control-center-brief.md. Section 6 rewritten with canonical invariant: "Status bar is identical in all three states. Same elements, same sizes, same positions across ambient, quick control, and full control. The ONLY difference is that engaged states add a status label bar seamlessly attached underneath."
>
> **Part 1: Status bar invariant** — ONE StatusBar struct used in ambient/quick/full. Takes showLabels: Bool. All sizing from HearthTypeSize/HearthLayout namespaces. Six elements left-to-right: Time (Instrument Serif large) + SUNDAY MORNING two-line right, outdoor temp (same scale as time) + conditions two-line right, Trinity (system→target, same size, no HERE), front door icon, garage icon, voice button. ALL temps at same Instrument Serif size.
>
> **Part 2: StatusLabelBar** — new component underneath status bar in engaged states. Geist Mono small, hearthCreamDim, tracked out, uppercase. Labels under each element. Animates in with slide-down.
>
> **Part 3: SUNDAY MORNING whimsy bridge** — day/period text appears in BOTH ambient and engaged. Never moves into label bar. Deliberate exception — bridges 10ft and 2ft viewing distances.
>
> **Part 4: Weather conditions with precipitation forecast** — outdoor element gains conditions text. Current condition + precipitation forecast 8h window. Positive affirmation: "NO RAIN EXPECTED" when nil. "RAIN AT 3PM" / "SNOW AT 3PM" / "WINTRY MIX AT 3PM". Note WeatherKit unavailable — use existing weather source.
>
> **Part 5: Quick control v3 (final form)** — strip everything. Top: canonical status bar + label bar. Body: downstairs lighting slider (power button left, slider, % label) + two nav icons (Shopping, Activity). Sparse. Kindle-like.
>
> **Part 6: Full control updates** — A: status bar replaced with canonical StatusBar+LabelBar. B: climate sub-line → full sentence "Heating toward 70° in the Kitchen." C: trinity HERE sub-label → "kitchen". D: lighting tiles 2x2 → 1x4 horizontal stack with inline sliders. E: lower-left nav icons get word labels. F: lower-right action tiles unchanged.
>
> **Part 7: What stays the same** — trinity arrow logic, mode pills, drill-in sheets, ECO behavior, living motion animations, photo frame, lower-right tiles, settings.
>
> **Part 8: Decision for CC — Kitchen label architecture** — three options. DC analysis points to Option 1 (PanelConfig.hereSensorLabel) but defer if it doesn't survive godthermostat.
>
> **Part 9: Voice button gameplan** — planning only, no implementation. Recommend AVSpeechSynthesizer vs pre-recorded audio. Give recommendation + reasoning + open questions.

---

## What Landed

### Part 1 + 2: StatusBar invariant + StatusLabelBar ✅

**`StatusBarView.swift`** fully rewritten. `StatusBar` is a single view struct used in all three states. `showLabels: Bool` parameter — when true, `StatusLabelBar` slides in underneath.

**Temperature invariant enforced:**
- Time: `statusClock` = 44pt ✅
- Outdoor temp: 34pt → 44pt ✅ (regression fixed)
- System temp: 36pt → 44pt ✅
- Target temp: 36pt → 44pt ✅

All four temps are now identical Instrument Serif size. No element larger or smaller.

**StatusLabelBar:** 28pt row (new `HearthStatusBar.labelBarHeight` token). Geist Mono labelSm, hearthCreamDim, 0.25em tracking, uppercase. Labels: TIME · OUTSIDE · SYSTEM/TARGET · FRONT DOOR · GARAGE · VOICE. Animates in with `.move(edge: .top).combined(with: .opacity)`.

**Trinity in status bar:** 2-temp only (system → target). HERE does not appear in the status bar. When mode is OFF: system temp only, no arrow, no target.

### Part 3: SUNDAY MORNING whimsy bridge ✅

Day + period labels (e.g. "SUNDAY" / "MORNING") appear as a two-line Geist Mono stack to the RIGHT of the time digits in ALL states — ambient and engaged. Never moves into the label bar. Documented as a deliberate exception in code comments:

> "SUNDAY MORNING — always present (the whimsy bridge)"

### Part 4: Weather conditions ✅

Open-Meteo already provides everything needed:
- `condition` string (from WMO code) → shown uppercase in conditions stack
- `precipitation` → `PrecipitationForecast` with `type` and `expectedTime`

**New `.mix` PrecipType:** Freezing rain / freezing drizzle WMO codes (56, 57, 66, 67) → `PrecipitationForecast(type: .mix, ...)`. View renders "WINTRY MIX AT 3PM".

**8-hour window:** `precipForecast` cutoff changed from 24h → 8h.

**Positive affirmation:** When `precipitation == nil`, the status bar renders "NO RAIN EXPECTED" in hearthCreamDim — not blank. This is the value: most apps only tell you about rain; we tell you affirmatively when you don't need a coat.

**Data source:** Open-Meteo (already working, no WeatherKit). Hourly precipitation probability gives everything needed. No backlog item — the full feature works today.

### Part 5: Quick control v3 ✅

Body-only view (StatusBar + StatusLabelBar live above in MainPanelView's VStack). Previous layout was stripped entirely.

**Upper body:** DOWNSTAIRS section label + HStack: power button (Ph.power, hearthAmber when on / hearthCreamDim when off) · brightness slider (full remaining width, tinted amber when on, dimmed when off) · percentage label (Instrument Serif quickAdjust).

**Lower body:** Shopping + Activity icons at HearthTouchTarget.minimum (80pt), tile background, no labels.

Design intent: sparse. No climate controls, no security tiles. The status bar already shows system→target arrow and door/garage indicators — quick control doesn't need to repeat them.

### Part 6: Full control updates ✅

**A.** StatusBar + StatusLabelBar wired into FullControlContent (same as quick control, same `showLabels: true`).

**B.** Climate sub-line: "Heating toward 70° · thermostat" → "Heating toward 70° in the Kitchen." Full sentence with period. Uses `PanelConfig.current.hereSensorLabel` for the location word.

**C.** Trinity HERE sub-label: `climate.sensorLocation` → `PanelConfig.current.hereSensorLabel` ("kitchen"). SYSTEM = "thermostat", TARGET = "what we set" unchanged.

**D.** RoomsSection: `LazyVGrid(columns: [.flexible(), .flexible()])` → `VStack` of full-width `RoomTile` instances. Each tile: `[Name (minWidth 88)] [Slider (maxWidth .infinity)] [75% label] [› chevron]`. Inline slider is display-only until 0d.2 writes land.

**E.** BottomActionRow left icons: unlabeled → `labeledIconBtn` with Geist Mono (labelSm−2pt) word labels. SHOPPING · ACTIVITY · HOME · BROADCAST · SETTINGS.

**F.** Lower-right action tiles (Front Door / Garage / Set Away / Awake) — unchanged per spec.

---

## Part 8: Kitchen Label Architecture Decision

**Decision: Option 1 — `PanelConfig.hereSensorLabel: String`**

**Reasoning:**

The HERE label names where *this panel's nearest sensor* is — a per-panel physical fact, not a HA configuration fact. It belongs in compile-time panel config, not in HA, because:

1. **Option 2 (rename HA entity)** conflates thermostat device identity with panel display context. The Nest's `friendly_name` shouldn't change because we want a panel label to say "kitchen."

2. **Option 3 (HA helper)** is correct architecture for things that change at runtime, but a sensor location doesn't change. An `input_text` helper per panel is infrastructure for a problem that doesn't exist.

3. **Option 1 survives godthermostat:** After the multi-sensor ESP32 upgrade, `hereSensorLabel` still names the sensor nearest to the panel. Whether it's the Nest in the kitchen or a godthermostat node in the kitchen, the label is the same. The code doesn't change — only the backing HA entity changes.

**What was built:** `PanelConfig` gains a `hereSensorLabel: String` field with default `"sensor"`. `firstFloor.hereSensorLabel = "kitchen"`. Used in `ClimateSection` sub-line ("in the Kitchen.") and trinity card HERE sub-label ("kitchen").

---

## Part 9: Voice Button Gameplan

**Recommendation: AVSpeechSynthesizer (Option A)**

**Reasoning:**

The panel's value proposition includes real-time dynamic data: current temps, current conditions, current mode. Pre-recorded audio (Option B) can't say "It's 73 degrees, heating toward 68, and it's foggy outside." It can only say fixed phrases. That's a ceiling.

AVSpeechSynthesizer on iOS 17 with a chosen Siri voice (specifically `com.apple.voice.enhanced.en-US.Evan` or similar — the "Jarvis-adjacent" voice Todd + DC identified) renders dynamic text naturally. Rate and pitch are tunable. No bundled audio files to maintain.

**What it would take:**

*Scope: ~2 hours, 2-3 files touched.*

1. **`VoiceService.swift` (new):** `@MainActor final class VoiceService`. `AVSpeechSynthesizer` instance. `func speak(_ text: String)` that creates `AVSpeechUtterance`, sets voice, rate (~0.50), pitch (~1.1 for slightly warmer tone), and calls `speak()`. Also a `func currentStatePhrase(home: HomeState, weather: WeatherState?) -> String` that assembles the spoken sentence.

2. **Audio session:** `AVAudioSession.sharedInstance().setCategory(.playback, mode: .spokenAudio)` once at init. No microphone permission needed for TTS.

3. **Wire into mic button:** The mic button in StatusBar currently calls `model.notifyWriteDisabled()`. Replace with `model.speakCurrentState()` which routes to `VoiceService`.

**Open questions before implementing:**

- Which specific Siri voice? Todd + DC should pick from `AVSpeechSynthesisVoice.speechVoices().filter { $0.language == "en-US" }` — there are ~6-8 options including enhanced (neural) voices on device.
- What phrases? A few templates to decide: "The house is [heating/cooling/idle] at [X]°, targeting [Y]°. Outside it's [Z]° and [condition]. The front door is [locked/unlocked]." Or shorter. DC should write the sentence — it sets the tone.
- First interaction: Cora. Should the phrase be short and child-friendly (temperature + one fun fact) or full status?
- Interrupt behavior: if voice is already speaking and the button is tapped again, should it cancel and restart, or queue? Restart is Jarvis-like.

**Option B case:** Pre-recorded files are strictly better if you want to control exact prosody, recording quality (a real mic vs iOS TTS), and the *specific inflection* on particular words. "The garage door is OPEN" said by a real voice with alarm inflection > TTS. A hybrid is viable: record 3-5 key alert phrases (garage open, front door unlocked) and use TTS for everything else. That's v2.

**My recommendation:** Ship AVSpeechSynthesizer for v1. Pick the voice, write the sentence template, implement in one focused phase. If the voice quality feels wrong after living with it for a week, revisit pre-recording.

---

## Broadcast Icon

`Ph.broadcast.bold` in `BottomActionRow` has no defined function. The `iconBtn` / `labeledIconBtn` helper renders it but there is no `onTapGesture`, no sheet, no action. It was vestigial from an earlier design concept.

**Flagging for DC:** Is Broadcast meant to represent HA's notification/broadcast service? HomePod speaker control? Something else? If it doesn't have a defined function after this phase, the cleanest option is to replace it with the fifth icon that DOES have a function, or drop to four labeled icons. DC to decide.

---

## Autonomous Design Decisions

- **`Ph.power` instead of `Ph.powerCircle`:** PhosphorSwift doesn't include `powerCircle`. Used `Ph.power` with color change (amber = on, creamDim = off) to encode state. Works cleanly.
- **1x4 room slider disabled:** `Slider` in the new room tile is `.disabled(true)` — display-only until 0d.2 writes land. A draggable slider that snaps back would be more confusing than a static one here, since the room tiles are primarily a drill-in affordance.
- **Label font size:** `labelSm − 2pt = 11pt` for the bottom row icon labels. labelSm (13pt) looked too heavy sitting under the 72pt icons. 11pt reads cleanly at engaged distance.
- **Quick control left-aligned icons:** Shopping and Activity icons are left-aligned (not centered). Centered looked odd in a sparse layout — left-aligns them with the DOWNSTAIRS section label above, giving the quick control body a consistent left edge.

---

## Commit Sequence

| # | SHA | Message |
|---|-----|---------|
| 1 | 0ac5941 | ui: canonical StatusBar + StatusLabelBar |
| 2 | aa5f596 | ui: wire StatusBar; rebuild quick control v3 |
| 3 | d1e824a | ui: weather 8h window, mixed precip, no-rain affirmation |
| 4 | b82c4e0 | ui: full control — 1x4 lighting, climate sentence, icon labels |
| 5 | 23a1d38 | release: bump to v0.5.0-ui-cleanup (build 50) |

Tagged: `v0.5.0-ui-cleanup`

---

## What Didn't Change (per spec)

- Trinity arrow logic (locked v0.3.8 — heating ↗, cooling ↘, idle ↔, off no arrow)
- Four mode pills HEAT/COOL/OFF/ECO
- Recursive lighting drill-in views
- ECO mode including midpoint logic
- Living motion animations (digit flip, brightness slide)
- Photo frame, photo date pill, swipe behavior
- Lower-right action tiles (Front Door / Garage / Set Away / Awake)
- Drill-in sheets
- Settings screen
- HAClient actor architecture

---

## Next Phase

**Phase 0d.2 — Write Implementation.** All 8 write method stubs on HAController throw `.notImplemented`. Implement them: setThermostatMode, setThermostatTarget, setEcoPreset (climate), setRoomBrightness, toggleRoom, setLightChild (lighting), toggleFrontDoorLock, toggleGarage (security), setHomeMode.

**Or: Voice Phase.** Wire the mic button to AVSpeechSynthesizer per Part 9 recommendation. Cora is the motivator. Scope ~2 hours once the sentence template is decided.

DC to prioritize.
