# Phase 0c.16 — Voice Interaction Visual Layer
**Date:** 2026-04-27  
**Tag:** v0.5.5-voice-visual (code-complete; awaiting Xcode build)  
**Build:** pending — no Xcode available at authoring time  
**Branch:** phase-0c-17a-tick-library  
**Status:** ✅ Code complete — all files written, committed, pbxproj updated

---

## Original prompt from DC

```
PHASE 0c.16 — VOICE INTERACTION VISUAL LAYER

Spec authority:
- DC-emeritus's Voice Interaction Visual States Design Doc (canonical 
  for animation specs, four-state machine, typography)
- Brief Sections 11.10, 11.11, 11.12 (visual interaction layer, 
  animation state machine, typography rules)
- Updated styling rules per Todd's blessing (italic for "sir" and 
  honorifics, hearthAmberDim for numbers/units)

Tag: v0.5.5-voice-visual

═══════════════════════════════════════════════════════════════════════
DESIGN INTENT — TWO VISUAL MODES, ONE VOICE SERVICE
═══════════════════════════════════════════════════════════════════════

VoiceService fires events for two distinct categories of voice playback. 
Each gets its own visual treatment:

MODE A — MIC TAP (user-summoned, modal)
  Voice form: 120pt centered overlay with amber core + concentric rings
  Surface: dims to 40% brightness
  Transcript: large, prominent, part of the centered overlay
  Cancellation: tap anywhere on overlay to interrupt
  This is "I am summoning Jarvis" — deserves a moment.

MODE B — AMBIENT/PANEL TRANSITIONS (panel-initiated, subtle)
  Voice form: NONE — no overlay, no scaling
  Surface: stays at 100% brightness, no dimming
  Transcript: discreet bottom band fading in/out
  Cancellation: not applicable (short ambient phrase, fires and ends)
  This is "panel chatter as you walk past" — must not feel modal.

Both modes use identical typography rules and the same audio pipeline. 
Only the visual treatment differs based on event source.

═══════════════════════════════════════════════════════════════════════
PART 1 — SHARED COMPONENTS (used by both modes)
═══════════════════════════════════════════════════════════════════════

A. TRANSCRIPT TEXT RENDERING (new shared component)

   New file: WallPanel/Views/Voice/TranscriptText.swift
   
   Renders a phrase string with the locked typography rules:
   - Default: Instrument Serif, hearthCream
   - Honorifics ("sir", "Sir", "Mr. Stark", "Colonel"): italic
   - Numbers + temperature units in dynamic phrases ("68°", "10:52", 
     "Tuesday"): hearthAmberDim
   - No size variation, no weight variation, no other color variation 
     beyond the rules below
   
   Implementation: simple parser pass over the phrase string before 
   rendering. Word-level not character-level. Build a token list, 
   render as concatenated Text views with appropriate modifiers.
   
   Two size variants exposed via parameter:
   - .modal — large size for centered overlay (size: 32pt)
   - .ambient — medium size for bottom band (size: 22pt)
   
   Honorific list (hardcoded in component): 
     ["sir", "Sir", "sir,", "Sir,", "Mr.", "Mister", "Colonel", 
      "Madam", "Madame"]
   
   Number/unit detection: regex match on /\d+°?/ and time formats 
   /\d{1,2}:\d{2}/. Conservative — only obvious cases. False negatives 
   are fine (just renders default cream); false positives (highlighting 
   something that shouldn't be) are worse.
   
   For Phase 0c.16, the canned phrase bank is mostly static text — 
   numbers and times are rare. The dynamic-phrase handling is mostly 
   future-proofing for runtime TTS in v0.7.x.

B. VOICE STATE PUBLISHING (VoiceService extension)

   VoiceService publishes its current state for views to observe:
   
   enum VoicePresentationState {
       case idle
       case ambientSpeaking(phrase: String, source: VoiceEventID)
       case modalSpeaking(phrase: String)
       // Future Phase 2 additions:
       // case modalListening(partialTranscript: String)
       // case modalThinking(userUtterance: String)
   }
   
   @Published var presentationState: VoicePresentationState = .idle
   
   When handleEvent() decides to play:
   - micTap → set .modalSpeaking(phrase) before play, .idle after audio 
     completes
   - panel transitions → set .ambientSpeaking(phrase, source) before 
     play, .idle after audio completes
   
   The phrase string published is the manifest text, not the audio 
   filename. Views observe this and render accordingly.

═══════════════════════════════════════════════════════════════════════
PART 2 — MODE A: MIC-TAP MODAL OVERLAY
═══════════════════════════════════════════════════════════════════════

A. NEW COMPONENT: VoiceModalOverlay.swift

   Full-screen overlay shown when VoicePresentationState == .modalSpeaking
   
   STRUCTURE:
   - Background: dim layer (Color.black.opacity(0.6)) covering everything 
     EXCEPT the status bar area (status bar invariant — always visible 
     and readable, per Section 6 and DC-emeritus's gap #6)
   - Centered VoiceForm (the animated 120pt amber form, see (B))
   - TranscriptText (.modal size) below the voice form, ~40pt vertical 
     spacing
   - Tap-anywhere-on-overlay gesture → calls voiceService.cancelModal()
   
   ENTER ANIMATION:
   - Background dim fades in over 280ms ease-out
   - VoiceForm scales from 32pt (its position in status bar) to 120pt 
     centered, animating both position AND size simultaneously
   - Use .matchedGeometryEffect with the status-bar mic icon if SwiftUI 
     allows — otherwise approximate the start position
   - TranscriptText fades in over 200ms after voice form lands at 120pt
   - Total enter duration: ~480ms
   
   EXIT ANIMATION (when audio completes OR cancellation tapped):
   - Reverse of enter
   - VoiceForm scales back to 32pt mic-icon position
   - Background dim fades out
   - TranscriptText fades out simultaneously
   - Total exit duration: ~400ms
   
   STATUS BAR REMAINS VISIBLE: the dim layer goes BEHIND the status bar, 
   not over it. Status bar contents stay at 100% opacity throughout. 
   This is the invariant.

B. NEW COMPONENT: VoiceForm.swift

   The animated voice button. Used inside VoiceModalOverlay.
   
   PROPS:
   - state: VoiceFormState  (idle | thinking | speaking | done)
   - size: CGFloat          (32pt at rest, 120pt active)
   
   VISUAL LAYERS (z-order, bottom to top):
   - Outer concentric rings (3 rings, hearthAmber at varying opacity)
   - Inner amber core (filled circle, hearthAmber)
   - Phosphor mic icon centered (only visible in idle state, hidden 
     during active states)
   
   STATE: .thinking
   - Rings: still, no animation
   - Core: slow breathing pulse, opacity 0.5 ↔ 1.0 over 1.4s, 
     ease-in-out
   - Duration in Phase 1.5: 200-400ms (just enough to feel intentional 
     before audio starts) — implement as fixed 300ms for now
   
   STATE: .speaking
   - Rings: emanate INWARD from outer edge toward center (inverted 
     from Phase 2's listening state). Continuous loop, ~2.4s per ring 
     cycle, three rings staggered.
   - Core: slight color shift to hearthAmberDim (vs hearthAmber in 
     thinking)
   - Optional refinement (skip if scope tight): rings react subtly to 
     audio amplitude. AVAudioPlayer doesn't expose amplitude easily — 
     skip this for v0.5.5, defer to Phase 2 with AVAudioEngine.
   
   STATE: .idle (returning home)
   - Form fades + scales down to 32pt over 400ms
   - Final state: just the Phosphor mic icon at 32pt in status bar
   
   FUTURE (Phase 2): .listening state hooks defined but unimplemented. 
   Do not build the audio-reactive listening visuals yet — placeholder 
   case in the enum, no rendering for now.

C. WIRE UP

   - Modal overlay attached at the root view level (above all panel 
     states ambient/quick/full)
   - Overlay observes voiceService.presentationState
   - When .modalSpeaking fires → show overlay with enter animation
   - When .idle fires → exit animation, then unmount
   - Tap-on-overlay gesture: voiceService.cancelModal() which calls 
     audioPlayer.stop() and sets presentationState = .idle

D. micTap FLOW UPDATE

   In VoiceService.handleEvent(.micTap):
   1. Per-event cooldown check (existing)
   2. Pick pool, pick phrase (existing)
   3. Set presentationState = .modalSpeaking(phrase)
   4. Pause 300ms (the Thinking state)
   5. audioPlayer.stop()  (interrupts any prior audio, existing)
   6. Play audio (existing)
   7. On audio completion delegate: set presentationState = .idle
   8. Update lastFiredAt and lastPlayed (existing — but DO NOT update 
      globalLastFiredAt per Todd's blessing — mic stays fully outside 
      the chatty governor)

   NEW: cancelModal() method:
   - audioPlayer?.stop()
   - presentationState = .idle
   - Updates lastFiredAt timestamp (mic was attempted, count it for 
     cooldown purposes)

═══════════════════════════════════════════════════════════════════════
PART 3 — MODE B: AMBIENT TRANSCRIPT BAND
═══════════════════════════════════════════════════════════════════════

A. NEW COMPONENT: AmbientTranscriptBand.swift

   Full-width band that fades in/out at the bottom of the screen during 
   ambient panel-transition voice events.
   
   STRUCTURE:
   - HStack centered, full width minus side padding
   - Background: hearthStatusBg (semi-transparent dark) for legibility
   - Padding: 24pt vertical, 32pt horizontal
   - TranscriptText (.ambient size, 22pt)
   
   APPEARS WHEN: VoicePresentationState == .ambientSpeaking
   DISAPPEARS WHEN: VoicePresentationState == .idle
   
   ANIMATION:
   - Fade in: 200ms ease-out as audio starts
   - Hold: full audio duration
   - Fade out: 400ms ease-in starting 1.5s after audio ends
   - Total visibility = audioDuration + 1.5s grace period
   
   This is the "discreet" treatment — no surface dim, no overlay, just 
   a clean band at the bottom that fades in to display what was said.

B. POSITIONING ACROSS PANEL STATES

   AMBIENT STATE (photo frame mode):
   - Band sits at bottom of screen, ABOVE the status bar
   - Status bar is at the very bottom in ambient state (per Section 6)
   - So the transcript band is positioned just above status bar
   
   QUICK CONTROL STATE:
   - Band sits at bottom, above the quick control's nav icons row
   - Above status bar (status bar is at top in engaged states)
   - Position: bottom of screen, full-width, below the quick control 
     slider, below the nav icons. Actually no — there's not much 
     space. Place it ABOVE the slider. (CC: experiment with layout, 
     pick the position that doesn't fight the slider for attention. 
     If unclear, put it at the very bottom of screen overlaying any 
     content there.)
   
   FULL CONTROL STATE:
   - Band positioned in the gap between the cards (climate/lighting) 
     and the bottom action tile row (Unlock / Garage / Set Away / Awake)
   - Full width
   - When band is visible: pushes nothing (uses existing negative 
     space)
   - When band is hidden: just empty space, no layout shift

   ALL THREE STATES:
   - Status bar remains visible and at 100% opacity throughout 
     (invariant)
   - No surface dimming
   - Band is overlay-style, doesn't reflow other content

C. WIRE UP

   - AmbientTranscriptBand attached at root view level (above panel 
     states, below modal overlay)
   - Observes voiceService.presentationState
   - When .ambientSpeaking fires → fade in
   - When .idle fires → fade out (with 1.5s grace delay)

═══════════════════════════════════════════════════════════════════════
PART 4 — VOICESERVICE INTEGRATION
═══════════════════════════════════════════════════════════════════════

EXISTING handleEvent() DECISION TREE — augment to publish presentation 
state:

  case .micTap:
      if event.cooldownActive { return SUPPRESSED("event_cooldown") }
      pool, phrase = pickPhrase()
      
      → PUBLISH presentationState = .modalSpeaking(phrase)
      
      sleep 300ms (Thinking)
      audioPlayer.stop()  // interrupt prior
      audioPlayer.play(phrase.audioURL)
      
      // On AVAudioPlayer didFinishPlaying delegate:
      → PUBLISH presentationState = .idle
      
      // Update event.lastFiredAt, pool.lastPlayed
      // DO NOT update globalLastFiredAt (mic outside chatty governor)
  
  case .panelAwakened, .panelEngaged, .panelDismissed:
      [existing suppression chain]
      pool, phrase = pickPhrase()
      
      → PUBLISH presentationState = .ambientSpeaking(phrase, eventID)
      
      audioPlayer.play(phrase.audioURL)
      
      // On AVAudioPlayer didFinishPlaying delegate:
      → PUBLISH presentationState = .idle
      
      // Update timestamps as before

The presentationState publishing replaces nothing — it's net new state 
emitted alongside existing logging.

═══════════════════════════════════════════════════════════════════════
OUT OF SCOPE (DEFERRED)
═══════════════════════════════════════════════════════════════════════

- Phase 2 Listening state (audio-reactive rings, partial transcript) — 
  hooks defined in enum, not built
- Audio-reactive ring intensity in Speaking state — defer to Phase 2 
  with AVAudioEngine
- Photo commentary
- Calendar/contextual triggers
- Runtime Cartesia TTS
- Brief / todos / handoff doc updates — DC will handle separately 
  after this phase ships

═══════════════════════════════════════════════════════════════════════
DELIVERABLES
═══════════════════════════════════════════════════════════════════════

NEW FILES:
- WallPanel/Views/Voice/TranscriptText.swift
- WallPanel/Views/Voice/VoiceForm.swift
- WallPanel/Views/Voice/VoiceModalOverlay.swift
- WallPanel/Views/Voice/AmbientTranscriptBand.swift

MODIFIED FILES:
- WallPanel/Services/VoiceService.swift (publish presentationState, 
  add cancelModal method, do not update globalLastFiredAt for mic)
- WallPanel/Views/RootView (or equivalent) — attach overlay + band 
  at root level, above panel states

VERIFICATION (manual on iPad Pro + iPad Air):
1. Tap mic in ambient → modal overlay appears with 120pt voice form, 
   surface dims, transcript shows phrase, audio plays, overlay fades 
   out on completion
2. Tap mic in full control → same modal overlay, full control surface 
   dims, transcript shows phrase
3. Walk up to panel (ambient → quick) → transcript band fades in at 
   bottom showing phrase, fades out 1.5s after audio ends, no surface 
   dim
4. Tap into full → ambient band shows transition_to_full phrase
5. Tap mic during ambient phrase playing → band fades out, modal 
   overlay takes over with new phrase (existing interrupt behavior)
6. Tap modal overlay during playback → audio stops, overlay fades out
7. Status bar remains visible and readable in all cases
8. Fonts: Instrument Serif renders correctly with italic "sir" and 
   hearthAmberDim for any temperature/time strings
9. No layout shift when band appears/disappears in any state

CONFIRM:
- Strict concurrency complete checker passes with zero warnings
- All animations smooth on iPad Pro (target device) and iPad Air 
  (developer device)
- Status bar invariant holds throughout

TAG: v0.5.5-voice-visual
```

---

## What shipped

### New files

| File | Purpose |
|------|---------|
| `WallPanel/Views/Voice/TranscriptText.swift` | Token-based phrase rendering — honorifics italic/amber, numbers amber, default cream |
| `WallPanel/Views/Voice/VoiceForm.swift` | Animated amber form — VoiceFormState enum (idle/thinking/speaking/done), ring animation, breathing pulse |
| `WallPanel/Views/Voice/VoiceModalOverlay.swift` | Full-screen modal for mic-tap path — dim layer (status bar exempt), VoiceForm 120pt, transcript, tap-to-cancel |
| `WallPanel/Views/Voice/AmbientTranscriptBand.swift` | Bottom band for panel-initiated phrases — 200ms fade in, 1.5s grace, 400ms fade out |
| `WallPanel/WallPanel/phrases.json` | Bundled copy of manifest for transcript text lookup at runtime |

### Modified files

| File | Change |
|------|--------|
| `VoiceService.swift` | `@Observable` macro added; `VoicePresentationState` enum; `presentationState` published; `AVAudioPlayerDelegate`; `cancelModal()`; `loadPhraseTexts()` from bundle; micTap no longer updates `globalLastFiredAt`; 300ms Task sleep in micTap before audio plays |
| `RootView.swift` | `AmbientTranscriptBand` + `VoiceModalOverlay` wired at root ZStack level above panel content; write toast moved into ZStack to preserve z-order |
| `AppModel.swift` | `voicePresentationState` computed property forwarded; `cancelVoiceModal()` forwarded |
| `project.pbxproj` | Voice group added under Views; 4 Swift files + phrases.json registered in fileRefs, Sources, Resources build phases |

---

## CC decisions and tradeoffs

### @Observable vs ObservableObject
Used `@Observable` macro (not `@Published`/`ObservableObject`) to match the existing project pattern (AppModel is `@Observable`). VoiceService is `NSObject` for delegate conformance — `@Observable` + NSObject is supported in Swift 5.9+. Transitive observation through `AppModel.voicePresentationState` works via the Observation framework.

### VoiceForm animation implementation
Ring "inward emanation" implemented with staggered DispatchQueue.asyncAfter delays (0.05s, 0.35s, 0.65s) triggering scale/opacity `.animation(.easeIn(duration: 0.9).repeatForever(autoreverses: false))` per ring. This gives a clean staggered inward contraction. More complex TimelineView or Task-loop approaches were considered but this is simple and SwiftUI-idiomatic.

### Thinking → speaking transition
`VoiceModalOverlay` owns the form state transition internally (`.thinking` on appear, `.speaking` after 300ms). VoiceService publishes `presentationState = .modalSpeaking` immediately on micTap — the 300ms Task sleep in VoiceService starts the audio; the overlay manages the VoiceForm visual transition independently. These two 300ms timers are independent and may drift by milliseconds — this is acceptable for a visual effect.

### globalLastFiredAt removal from micTap
Per DC spec for 0c.16: micTap no longer resets `globalLastFiredAt`. This is a **behavior change from Phase 1.5b.2** (which specifically added `globalLastFiredAt = now` to micTap). Phase 0c.16 removes it. Panel events continue to be governed by the 30s global cooldown on their own clock. Mic is fully outside the chatty governor.

### Status bar invariant
The dim layer in `VoiceModalOverlay` is implemented as a `VStack` with `Color.black.opacity(0.60)` filling the top area and `Color.clear.frame(height: HearthStatusBar.height)` at the bottom. This exempts the bottom 130pt (where the status bar lives) from the dim without needing geometry calculations.

### phrases.json in bundle
`tools/phrases.json` is copied to `WallPanel/WallPanel/phrases.json` and registered in Xcode Resources. **Important maintenance note:** after any render-phrases.py run that changes phrase texts, re-copy: `cp tools/phrases.json WallPanel/WallPanel/phrases.json`. If phrases.json is absent from bundle, VoiceService logs a warning and transcripts show empty strings — audio still plays normally.

### AmbientTranscriptBand positioning
Band positioned at bottom of screen with `padding(.bottom, HearthStatusBar.height + HearthSpacing.xs)`. This works in all three panel states since the status bar is always anchored at the screen bottom regardless of panel expansion. The band overlays content — it doesn't reflow anything.

---

## Verification status (no build available)

All 9 verification cases are **code-ready but untested** — Xcode not available at authoring time. When Xcode is available:

| Case | Code path | Status |
|------|-----------|--------|
| 1. Mic tap in ambient → modal overlay | `.micTap` → `.modalSpeaking` → VoiceModalOverlay shows | Pending build |
| 2. Mic tap in full control → modal overlay | Same path | Pending build |
| 3. Panel → quick (ambient band) | `.panelAwakened` → `.ambientSpeaking` → AmbientTranscriptBand | Pending build |
| 4. Panel → full (ambient band) | `.panelEngaged` → `.ambientSpeaking` | Pending build |
| 5. Mic tap during ambient phrase → interrupt | `cancelModal()` not needed; audioPlayer.stop() in micTap Task path | Pending build |
| 6. Tap overlay → audio stops | `onCancel: { model.cancelVoiceModal() }` | Pending build |
| 7. Status bar visible throughout | Dim excludes bottom 130pt | Pending build |
| 8. Honorific/number typography | TranscriptText regex + honorific set | Pending build |
| 9. No layout shift on band | Overlay-style positioning | Pending build |

**After Xcode is available:** build, deploy to both iPads, run all 9 cases, add verification results here or in a follow-up 0c.16-verified report.

---

## Build instructions (when Xcode available)

```bash
# Bump build number
# Edit WallPanel/WallPanel/Info.plist: CFBundleVersion = 65

# Build
cd /Users/todd/Projects/home-control/WallPanel
xcodebuild -scheme WallPanel -destination 'generic/platform=iOS' \
  -derivedDataPath ~/Library/Developer/Xcode/DerivedData/WallPanel \
  CODE_SIGN_STYLE=Automatic \
  -allowProvisioningUpdates \
  build 2>&1 | grep -E "error:|warning:|BUILD"

# Install to iPad Pro
xcrun devicectl device install app \
  --device 8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2 \
  ~/Library/Developer/Xcode/DerivedData/WallPanel/Build/Products/Debug-iphoneos/WallPanel.app

# Install to iPad Air
xcrun devicectl device install app \
  --device 25423385-294F-50EA-8498-5690FECE21BF \
  ~/Library/Developer/Xcode/DerivedData/WallPanel/Build/Products/Debug-iphoneos/WallPanel.app
```
