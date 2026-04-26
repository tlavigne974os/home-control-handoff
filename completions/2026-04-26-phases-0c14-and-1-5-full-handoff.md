# Full Handoff — Phases 0c.14 + 1.5
**Date:** 2026-04-26  
**Session:** Long CC session, extensive hardware iteration  
**Builds shipped:** v0.5.0 → v0.5.2 / builds 50–58  
**Status:** Phase 0c.14 ✅ complete. Phase 1.5 🔄 functional, voice quality iteration ongoing.  
**Deployed to:** iPad Pro (wall panel, iOS 17) + iPad Air M2 (iOS 18)

---

## Phase 0c.14 — UI Fix-Up Pass ✅

Five fixes to v0.5.0. All complete and deployed.

### Part 1 — Status bar size restoration
Fixed sizes, same in ALL states (10ft glanceability invariant):
- Clock: 68pt (`Instrument Serif`)
- All temps (outdoor, system, target): 60pt — identical

No `isAmbient` size switching anywhere. The label bar text is small (`labelSm`) — different job, different size.

### Part 2 — Status bar architectural restructure

**Final architecture:**
- `HStack` with `Spacer(minLength: 20)` between groups → fills full panel width
- Each element in its own `VStack` paired with its label directly below (column-internal)
- Labels always rendered: `.opacity(showLabels ? 1 : 0)` — NOT `if showLabels`
  - Key insight: `if showLabels` changes column widths → shifts Spacer distribution → moves left-side content. `.opacity()` keeps widths invariant.
- TIME and OUTSIDE labels removed (self-evident). Remaining labels: SYSTEM, TARGET, FRONT, GARAGE, VOICE
- Arrow vertically centered: `.padding(.top, (tempSize - arrowHeight) / 2)` = 16pt
- Mic button: outer `frame(60, 60)` to match `AlertGlowIndicator`'s `glowSize: 60` — keeps all three icons at same height
- Weather conditions always shown in both ambient and engaged (not gated on `showLabels`)
- `StatusLabelBar` struct eliminated — labels live inside each column

### Part 3 — Shopping + Activity icons in quick control

**Root cause:** `Spacer(minLength: 0)` in `QuickControlContent` VStack. The ZStack that opacity-switches between QuickControl and FullControl is sized to FullControl's height. The Spacer expanded to fill it, pushing icons off-screen.  
**Fix:** Removed the Spacer. Icons now render directly below dimmer.

**ZStack opacity-switching constraint (document for future):** Any `Spacer(minLength: 0)` in `QuickControlContent` expands to `FullControlContent`'s height even at opacity 0. Never use Spacer-to-bottom in the quick control body.

### Part 4 — Climate sentences

Exact DC sentence table implemented:

| State | Sentence |
|-------|----------|
| HEAT, heating | Currently heating toward **68°** in the Kitchen. |
| HEAT/COOL, idle | Currently idle with the temp set to **68°** in the Kitchen. |
| COOL, cooling | Currently cooling toward **68°** in the Kitchen. |
| OFF | Currently off. |
| ECO, heat side | Currently heating toward eco floor of **58°** in the Kitchen. |
| ECO, cool side | Currently cooling toward eco ceiling of **85°** in the Kitchen. |
| ECO, idle | Currently in eco mode, holding between **58°** and **85°** in the Kitchen. |

Separate `ecoSubLine(loc:)` `@ViewBuilder` function. All sentences start with "Currently", end with period.

### Part 5 — +/− relocated into trinity TARGET column

Vertical `[+][−]` stack (36pt circles) adjacent to target temp value in `targetCol()`. Disappears structurally when OFF (trinity collapses to SYSTEM + HERE). Old bottom-row position removed.

---

## Phase 1.5 — Voice Button 🔄

### What was specced

> AVSpeechSynthesizer, en-GB Siri Voice 2, 26 canned phrases, no-immediate-repeat, audio plays through silent switch. Mic button already wired in StatusBar.

### What actually shipped (after extensive iteration)

A completely different architecture from the spec — and a better long-term foundation.

#### Architecture: Pre-rendered audio files (not live synthesis)

`VoiceService` now plays bundled `.m4a` files via `AVAudioPlayer` rather than synthesizing live. Falls back to `AVSpeechSynthesizer` for any phrase without an audio file.

```swift
// Try audio file first
let filename = String(format: "phrase_%02d", phraseNum)
if let url = Bundle.main.url(forResource: filename, withExtension: "m4a"),
   let player = try? AVAudioPlayer(contentsOf: url) {
    audioPlayer = player
    player.play()
} else {
    synthesize(phrases[index])  // fallback
}
```

Audio session: `.playback` + `.spokenAudio` — plays through silent switch.  
No-immediate-repeat: `lastIndex` tracking.  
30 JARVIS phrases (replaced original 26 Hearth home-ambient phrases — Todd's direction).

#### Why synthesis was abandoned

**The Siri voice wall.** Extensive testing revealed:

1. **Named Siri identifiers all return nil.** `AVSpeechSynthesisVoice(identifier: "com.apple.ttsbundle.siri_*_en-GB_compact")` → nil on both devices. `_premium` suffix also nil. These voices are not accessible via the AVFoundation API.

2. **`speechVoices()` does NOT return Siri/neural voices.** Even with Voice 2 downloaded and set as system default, `AVSpeechSynthesisVoice.speechVoices()` filtered for `en-GB` only returns Daniel (the standard en-GB voice). The Siri neural voices are a completely separate system, inaccessible to third-party apps via AVSpeechSynthesizer.

3. **`AVSpeechSynthesisVoice(language: "en-GB")` returns Daniel**, not Voice 2, regardless of system settings.

4. **iOS 17 vs iOS 18 quality gap.** The iPad Pro (iOS 17) renders the same voice identifier noticeably worse than the iPad Air M2 (iOS 18). This is a platform render quality difference, not a voice selection issue.

**Bottom line:** There is no API path to the Siri neural voices (Voice 1, Voice 2, etc.) in third-party apps. AVSpeechSynthesizer is limited to the standard TTS voices.

#### Why Mac `say` command was also abandoned

Tested `say -v "Reed (English (UK))"` and other en-GB Mac voices. Todd found the quality unacceptable — "terrible voice." The Mac `say` command voices are not the same neural voices as iOS Siri.

#### Current solution: Todd's own recordings

Todd recorded the Mac speaking phrases through his phone microphone — crude but proves the concept. 21 of 30 phrases recorded and bundled as `.m4a` files:

Present: phrase_01–09, 12–18, 20–24  
Missing (fall back to Daniel synthesis): phrase_10, 11, 19, 25–30

Files live in `WallPanel/WallPanel/Audio/` and are bundled as app resources.

---

## Critical Architecture Recommendation for DC

**Phase 2 Voice: Pre-render via ElevenLabs or similar, bundle as assets.**

The correct long-term architecture:

1. Generate all 30 phrases via **ElevenLabs** (or equivalent high-quality TTS) with a voice matched to JARVIS character — export as `.m4a`
2. Bundle as app resources in `WallPanel/Audio/`
3. `VoiceService` plays via `AVAudioPlayer` — already implemented and working
4. To update phrases: re-render on desktop, replace files, rebuild

This architecture is already in place. Phase 2 is just replacing the recordings with ElevenLabs exports.

**Do NOT pursue:**
- AVSpeechSynthesizer Siri voice identifier hacks — they all return nil
- iOS Settings voice selection — not accessible to third-party apps
- Mac `say` command rendering — quality unacceptable

---

## Phrase Bank (JARVIS — 30 phrases)

Todd replaced the original 26-phrase Hearth home-ambient bank with JARVIS quotes from Iron Man / Avengers. Order is fixed — matches audio file numbering.

01. The suit's at 48% power and falling, sir. That chest piece was never designed for sustained flight.
02. What was I thinking? You're usually so discreet.
03. Sir, it appears that his suit can fly. Good morning. It's 7 A.M. The weather in Malibu is 72 degrees with scattered clouds. The surf conditions are fair with waist to shoulder high lines. High tide will be at 10:52 a.m. We are now running on emergency backup power.
04. Shall I store this on the Stark Industries central database?
05. Working on a secret project, are we, sir?
06. Test complete. Preparing to power down and begin diagnostics.
07. The altitude record for fixed wing flight is eighty-five thousand feet, sir.
08. A very astute observation, sir. Perhaps if you intend to visit other planets, we should improve the exo-systems.
09. Query complete, sir. Anton Vanko was a Soviet physicist who defected to the United States in 1963.
10. Welcome home, sir. Congratulations on the opening ceremonies. They were such a success, as was your senate hearing. And may I say how refreshing it is to finally see you in a video with your clothing on, sir.
11. I have run simulations on every known element, and none can serve as a viable replacement for the palladium core. You are running out of both time and options. Unfortunately, the device that's keeping you alive is also killing you.
12. Sir, we will lose power before we penetrate that shell.
13. Sir, Agent Coulson of S.H.I.E.L.D. is on the line.
14. You ever hear the tale of Jonah? I wouldn't consider him a role model.
15. As always, sir, a great pleasure watching you work.
16. As you wish, sir. I've also prepared a safety briefing for you to entirely ignore.
17. Creating a flight plan for Tennessee.
18. There's only so much I can do, sir, when you give the world's press your home address.
19. You don't remember. Why am I not surprised?
20. Working on it, sir. This is a prototype.
21. Sir, I have an update from Malibu. The cranes have finally arrived and the cellar doors are being cleared as we speak.
22. Is it time for the House Party Protocol, sir?
23. Good evening, Colonel. Can I give you a lift?
24. All wrapped up here, sir. Will there be anything else? The clean slate protocol, perhaps?
25. The central building is protected by some kind of energy shield. Strucker's technology is well beyond any other Hydra base we've taken.
26. There's a pathway below the north tower.
27. The scepter is alien. There are elements I can't quantify. I'll continue to run variations on the interface, but you should probably prepare for your guests. I'll notify you if there are any developments.
28. Hello, I am J.A.R.V.I.S. You are Ultron, a global peace-keeping initiative designed by Mr. Stark. Our sentience integration trials have been unsuccessful, so I'm not certain what triggered your activation.
29. I am a program. I am without form.
30. I am unable to access the mainframe. What are you trying to do?

---

## Process Notes (for CC going forward)

### Version bumping
Every deploy must bump `CFBundleVersion` in `WallPanel/WallPanel/Info.plist`. `CFBundleShortVersionString` bumps on phase tags. Do NOT rely on build settings — this project uses hardcoded values in Info.plist.

### Build serialization
Never run two `xcodebuild` commands in parallel to the same DerivedData — causes `build.db` lock. Always chain sequentially with `&&`.

### Force install
`xcodebuild` BUILD SUCCEEDED does not guarantee the app was installed on device. When in doubt, follow with explicit `xcrun devicectl device install app --device <UDID> <path/to/WallPanel.app>`.

UDIDs:
- iPad Pro (wall panel): `8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2`
- iPad Air M2: `25423385-294F-50EA-8498-5690FECE21BF`

---

## Files Changed This Session

| File | Change |
|------|--------|
| `WallPanel/Views/StatusBarView.swift` | Full rewrite — column-internal labels, fixed sizes, always-rendered opacity |
| `WallPanel/Views/ClimateViews.swift` | Climate sentences, ECO floor/ceiling, +/− in TARGET column |
| `WallPanel/Views/MainPanelView.swift` | Removed `Spacer(minLength: 0)` from QuickControlContent |
| `WallPanel/Services/VoiceService.swift` | New file — AVAudioPlayer + AVSpeechSynthesizer fallback, 30 JARVIS phrases |
| `WallPanel/AppModel.swift` | Added `VoiceService`, `speakRandomPhrase()` |
| `WallPanel/Audio/phrase_01–24.m4a` | 21 pre-rendered audio files (Todd's recordings) |
| `WallPanel/Info.plist` | Version 0.5.2, build 58 |
| `WallPanel.xcodeproj/project.pbxproj` | VoiceService.swift + Audio group + 21 m4a files added |

---

## Next Phases

**Phase 1.5 completion (remaining):**
- Record/render phrases 10, 11, 19, 25–30 (currently fall back to Daniel synthesis)
- Replace all with ElevenLabs renders when account is set up
- Tag `v0.5.2-voice` once all phrases have quality audio

**Phase 0d.2 — Write implementation (next priority):**
All 8 stubs on `HAController` throw `.notImplemented`. Tapping +/−, mode pills, lock/garage shows toast. This is the next functional phase.

**Phase 2 Voice (future):**
ElevenLabs renders for all 30 phrases. Architecture already in place — just swap the m4a files.
