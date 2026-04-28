# Home Control Center — Independent Audit (AUC, fresh-context Opus)

**Date:** 2026-04-28
**Auditor:** AUC — fresh-context Claude Opus, no prior project exposure
**Build reviewed:** 71 (compiled, on iPad: 70)
**Materials read in this order:** brief → todos → handoff doc → DC strategic audit → CC code-health audit v2 → six most recent completion reports → spot-checks of `VoiceCoordinator.swift`, `baxter-system-prompt-v2.md`, git log on app and handoff repos.

This audit is the third pass. CC has the code right. DC has the design intent right. My job is to look at what *both* have accepted as obvious that an outsider should question.

---

## 1. First impressions, cold

Walking into this project, three things hit before I read any other audit:

The brief is unusually good. Most personal-project specs over-promise on principles and under-specify on tradeoffs. This one does the opposite: 17 principles that mostly earn their keep, lots of explicit "we chose X because Y, here's the swap criterion if Y stops being true." The principle structure is doing real work — design decisions are constrained by it, not just narrated.

The voice subsystem dwarfs everything else in the project. Twelve files in `Services/Voice`, ~2,000 lines of pipeline orchestration code, four external dependencies (Anthropic, Cartesia, Picovoice, Apple), three abstractions (`VoiceAssistant`, `CartesiaWSClient`, `AudioStreamPlayer`), and a versioned system-prompt archive with a sync script and a public mirror repo. Compare that to the actual home-control surface area — eight write methods, all stubs that throw `notImplemented`. The panel that is "always 100% capable" (Principle 10) currently controls zero entities. Voice that is supposed to handle the 20% currently handles 0%.

The development pace is faster than the verification pace by roughly an order of magnitude. Build 71 was compiled today; build 70 is on the iPad. Six builds in 36 hours. None of the recent four phases have been verified on device. The "USB cable not available" rationale appears in three consecutive completion reports.

These three observations are connected. I'll come back to that.

---

## 2. Where I agree with CC

CC's code-health audit is solid. I'll just confirm and move on:

- **Stage 4 tick gating is real.** I read `VoiceCoordinator.swift` lines 225–248 and the gate is exactly what CC describes — `await awaitAVAudioPlayer(tickPlayer)` blocks `audioPlayer.start()` even if Cartesia chunks are already buffered. Build 70's "feels faster" experience is the no-op-Stage-4 path. CC's fix (`tickPlayer?.stop()` then start) is correct and trivial.
- **CartesiaWSClient is rebuilt every turn.** Confirmed at line ~157.
- **The five latent bugs are real** (empty transcript, concurrent pipeline, awaitCompletion hang, wake-word silent death, continuation leak). All five surface naturally once wake word activates.
- **Telemetry prefix unification is incomplete.** `──` prefixes still in `CartesiaWSClient.swift` and `AudioStreamPlayer.swift`.

CC's prioritized fix list is the right list at the right granularity. I have nothing to add to it on the code level.

---

## 3. Where I agree with DC

DC's strategic frame holds up:

- **Verification gap is the dominant project risk.** I'd put this stronger than DC does — see §5.
- **Layer 1/Layer 2 ordering is conceptually right.** The principle is good; the application has slipped.
- **Principle 15 is drifting.** Voice now has four external dependencies, three of them paid/limited.
- **The Hearth design system, the actor-based HAClient, and the completion-report archive are genuine strengths.** Don't disturb.

I'll skip the "character operating on a ghost" item per the prompt.

---

## 4. Where I disagree

**With DC: "silent grace" should NOT be promoted to a design principle.**

DC suggests promoting the emergent silent-failure pattern to Principle 18: "When voice cannot fulfill a request, it does not speak rather than apologize." This sounds elegant and serves the character. It is wrong for this project.

Todd is deaf. The voice character work serves Mei-Ling and Cora. When Mei says "Baxter, turn off the kitchen lights" and the network is down, "silent grace" produces an experience that reads as: *did the panel hear me? Did Baxter break? Should I ask again?* That is the opposite of what the visual-feedback system is supposed to provide. The TranscriptBand exists precisely so failures are visible. A pattern that suppresses both audio and transcript on failure has stripped its own accessibility surface in service of character coherence.

The right rule is the inverse: **failures are always visible** (as transcript text, even if no audio plays), and the *content* of the failure message is in-character. "I didn't catch that, sir." is in-character and informative. Pure silence is in-character and useless. CC noted the empty-transcript bug; this is the same problem, generalized.

**With CC and DC: the v2 system prompt isn't just "Woadhouse residue." It's actively contradictory.**

CC flagged the "You are Woadhouse" identity paragraph as a high-severity issue. That's the visible part. The deeper issue is that the v2 prompt contains *both* backronyms in sentences a few lines apart:

- NAME HANDLING (line ~96–100): backronym is **"Baritone Aide of Xenial Temperament, Ever Ready"** — and the assistant should answer **"Baxter, sir."**
- Examples block (line ~118–120): `Todd: "What does Woadhouse stand for?" / You: "Watchful Observer And Diligent Helper Of Unparalleled Service, Endlessly. Bit much, I know."`

Claude Haiku is being told two different names, two different backronyms, and shown a worked example modeling the *old* name and *old* backronym. The example takes precedence over the rule — that is how few-shot prompting works. So even though NAME HANDLING is correct, the worked example is teaching the model to answer with the Woadhouse backronym. v3 needs a full sweep, not an identity-paragraph fix.

**With both: the development cadence question is treated as a verification problem when it's actually a steering problem.**

DC frames the cable-availability bottleneck as "accumulated risk." CC names it as a known issue. Both are right that lived use catches bugs. Both are missing what the bottleneck *is structurally*. Six builds in 36 hours, executed by CC, planned by DC, with no time for Todd to live with each one — that means there is no out-of-loop signal for course correction. The audit framework (CC + DC + AUC) catches what code review catches and what design review catches. It does not catch what *use* catches. Use is the missing input. The cable is just where the missing input shows up.

I'll come back to this in §7.

---

## 5. Things both audits missed

**5.1 Wake word is listed as a non-goal in the brief.**

Brief Section 17, item 1: *"Wake-word voice activation (tap-to-talk is the choice)"*. The brief is described in its own header as the *canonical reference*. Phase 0c.19, currently the next phase on the roadmap with three Todd preconditions queued, is implementing wake word. Neither audit caught this. The brief's voice section was updated for 0c.19 but the non-goals section was not. There is no recorded decision flipping this non-goal — no completion report explains *why* tap-to-talk stopped being "the choice."

This may be a small oversight in doc maintenance. It may also be a sign that the roadmap is now driving the brief instead of the brief driving the roadmap. Worth knowing which.

**5.2 Todd cannot evaluate Baxter's audible character.**

Both audits treat "is the character right" as something Todd's lived use will answer. Todd is deaf. He often doesn't wear hearing aids at home. He can read the system prompt, read the transcript, and read the LLM's textual output — all of which he has been doing. He cannot hear Benedict's voice, the 0.8x speed, the measured tempo, the British register, or whether the acknowledgment-phrase + thinking-tick + response stack sounds graceful or stuttery in audible succession.

Baxter's *textual* character is calibrated against the LLM. Baxter's *audible* character is downstream — Cartesia voicing whatever the LLM produces. The textual calibration (RESTRAINT, vocabulary discipline, "Indeed sir") is what Todd sees. The audible experience (does the tick handoff sound like one event or two? does the voice feel hurried? does Cora find it grating?) is what Mei and Cora hear and Todd does not.

CC writes that "Todd's lived experience of a calibrated Baxter character in builds 67–70 was real and accurate." That is true *for the textual layer*. It is undefined for the audible layer. The character work to date has been calibrated almost entirely against the layer Todd can perceive, which is a partial signal. Mei and Cora's reactions are the missing data, and neither audit asks whether they have been collected.

**5.3 The Vesper → Woadhouse → Baxter renames happened with no audible test.**

Three names in two days, all chosen from prompt-design level. Each rename came with a stated rationale ("Woadhouse was always too much"). None came from "I heard Cartesia render this name three times and it doesn't work." The character-thrash question DC asked secondary-7 deserves a sharper answer: this is character thrash, because the test of a wake-word name is what it sounds like coming out of Benedict, and that test has not been run.

**5.4 "Wake word" + "name said by family members" + "Picovoice trains a custom word" is a coupled decision that wasn't framed as one.**

Picovoice's training tool has finite phonetic discrimination. "Baxter" is a common-ish word; the brief flags "validate phonetic strength, fall back to Hey Baxter." The wake-word phonetic test, the family-says-the-name ergonomics test, and the Cartesia-voice ergonomics test are three *different* tests of the same name. No completion report or audit shows them being evaluated together. The risk is choosing a name that passes one test and fails the others, then having to rename a *fourth* time with infrastructure already wired around it.

**5.5 The thinking-tick is layered on top of the acknowledgment-phrase pattern, and nobody asked if both are needed.**

The acknowledgment phrase ("Indeed, sir.") was added in v1 system prompt to mask LLM TTFT — it ships through TTS while the substantive answer is still generating. The thinking tick was added in 0c.18 to mask the *same* LLM TTFT. Both fire on every conversational turn. Both are designed to fill silence during LLM latency. From the user's POV, every interaction now opens with: *(tick: "Mm. Let me see...") (response: "Indeed, sir. Just past ten.")*. That is a stall-prefix layered on a stall-prefix.

Either one alone covers the latency gap when streaming is working. Together they may produce a feeling that Baxter is *always* hesitating. CC's audit identifies the tick as a possible latency penalty when it runs longer than LLM+TTS. I think there's a deeper question: with streaming working, do we even need a tick library? The acknowledgment phrase already exists and is character-aligned. The 30-phrase tick library is implementation cost protecting against a problem the streaming pipeline already solves.

**5.6 The cable bottleneck is a tooling problem with a 10-minute fix, not a scheduling problem.**

DC notes the cable as "USB connection has been a recurring blocker." CC's audit doesn't address the deploy chain. Three completion reports mention "USB unavailable." `xcrun devicectl` supports `--device <UDID>` over the local network after a device has been paired once over USB. iPad Pro at UDID `25423385-294F-50EA-8498-5690FECE21BF` and iPad Pro M2 at `8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2` are both listed as paired. Setting up wireless deploy is a one-time configuration step. The project has rationalized a tractable problem into a recurring bottleneck.

This matters because the cable bottleneck *is* the verification gap.

**5.7 Voice has no offline degradation path.**

Principle 15 says "every external dependency must answer: what happens when this is gone?" The voice subsystem has four — Anthropic (LLM, paid), Cartesia (TTS, paid), Picovoice (wake word, free for personal), Apple (STT, free, local). If Anthropic is rate-limited or down: voice is dead. If Cartesia changes pricing or the WebSocket endpoint breaks: voice is dead. If Picovoice changes its "free for personal" terms: wake word breaks. The "silent grace" pattern means none of these failure modes produce useful feedback — they all fail to the same outcome (no response, no transcript). Cora asks Baxter for a story; nothing happens; no signal as to why. The system has no fallback to bundled phrases for "I'm currently unavailable, sir" — even though 199 bundled phrases exist and this is exactly the kind of situation that the canon_main pool could be repurposed for.

**5.8 The Layer 1/Layer 2 split has a more fundamental issue than DC's "polished vs built."**

DC says Layer 1 is "polished" only after lived verification. That's right but doesn't go far enough. Layer 1 as currently scoped is "the experience of talking to a thing that can't do anything." Layer 2 is "the experience of talking to a thing that *can* do something." These are different products. The deflection patterns ("my hands are largely metaphorical, sir") are Layer 1 fixtures that get retired in Layer 2. The acknowledgment cadence appropriate for capable-Baxter may be different from incapable-Baxter. The wit-restraint discipline calibrated against deflection-heavy interactions may need re-calibration for command-response interactions.

Polishing Layer 1 fully before Layer 2 risks calibrating against a transient state. Some Layer 1 polish *will* survive Layer 2 (Hearth visuals, acknowledgment pattern, address conventions). Some won't (deflection patterns, the entire "What you cannot do" section of the system prompt). The brief doesn't make this distinction; the audits inherited it as if Layer 1 were a stable target.

**5.9 The example in v2 system prompt now teaches Claude to refuse a story.**

Line 135–137: `Todd: "Tell me a joke." / You: "Mm. The thermostat goes into a bar — but I'm not yet wired in to know what it orders. Ask me again next month."`

The new LENGTH section of v2 says explicitly: *"Do not refuse stories."* Yet the worked example (which carries more prompt-engineering weight than the rule) shows Baxter refusing a joke by leaning on the deflection pattern. This is the same kind of contradiction as the Woadhouse-vs-Baxter backronym. The v3 sweep needs to update worked examples to model what the *new* rules are asking for, not the old behaviors.

**5.10 The "Cora and Mei are the true users" claim from the handoff is undefended.**

Handoff doc final notes: *"Cora and Mei are the true users. Todd is shipping this for the household."* Searching the docs for "Cora" or "Mei-Ling" outside lists of household members and the system prompt: the references are sparse, and almost none describe what Cora or Mei have actually done with the panel or said about it. The "would Cora find this delightful?" test is invoked rhetorically but not run. The character vision (dry-British-butler), the wake-word name, and the address conventions ("Miss Cora") were all designed without Cora in the loop, as far as the documentation shows.

This is the single biggest gap in the project's evidence base. It is invisible from inside the loop because the loop doesn't include Mei or Cora.

---

## 6. The thing nobody is saying

The project is building a voice-character infrastructure for an audience of people who haven't been consulted, evaluated by a primary user who can't hear it, on a cadence that doesn't allow lived testing, with a "silent grace" failure mode that ensures problems won't surface until a family member gets confused enough to mention it.

That's the thing. Both prior audits orbit it without naming it. CC says character was real all along; DC says verification gap is dominant; the project narrative is "Baxter is becoming who he needs to be." The honest read is closer to: **the voice character is being designed by Todd-and-his-Claudes for an idealized version of Mei and Cora, and the iteration will not converge on what Mei and Cora actually want until Mei and Cora actually use it.**

The fix is not more character calibration. It is two days of Mei and Cora using build 71 with notes-collection that gets back to the project. Until that happens, every system-prompt iteration is a guess.

---

## 7. The three-Claude-plus-Todd structure

Honest read: **the structure is producing high-quality artifacts and lower-quality outcomes than the artifact quality suggests.**

The artifacts are excellent. Brief, todos, handoff, completion reports, audits — this is better project documentation than most professional teams produce. The character versioning and sync script are genuinely thoughtful. The handoff repo as memory infrastructure works.

The outcome question is different. Six builds in 36 hours, none verified on device, two material bugs (Stage 4 tick gating, Woadhouse residue) that lived use would have caught in five minutes. The system has a velocity, but the velocity is decoupled from a feedback signal.

Here's the structural failure mode: DC writes prompts. CC executes prompts. CC writes completion reports. DC reads reports and writes next prompts. AUC (me) audits the whole thing. *None* of these inputs are "Mei used the panel and said X" or "Cora pushed the mic and the response surprised her." The Claude loop closes on itself; the only out-of-loop signal is Todd, and Todd is one person managing three Claudes plus the rest of his life. When he's tired ("don't argue, just do it"), the steering signal goes to zero and the loop runs open-loop.

The "don't argue, just do it" pattern is worth examining. It's been documented as a feature ("when Todd says don't argue, drop pushback and execute"). DC has codified it. CC respects it. AUC inherits it. From inside the loop, this looks like accommodation of a busy collaborator. From outside, it looks like the steering signal getting weaker at exactly the moments when it's most needed — when the human is tired and the AIs have momentum.

**Concrete: the structure is good. The artifacts it produces are good. But it has a steering deficit and the deficit is not visible from inside.** The fix is not "fewer Claudes" or "different process." The fix is widening the input. Mei and Cora need to be in the loop, not as recipients but as evaluators. Otherwise the next thirty completion reports will look like the first thirty.

The pause-to-audit moment is itself the right move. Hold it. Don't immediately resume velocity.

---

## 8. If I were Todd

In priority order. Each is a concrete next action.

**1. Set up wireless device deployment today (10 minutes).** `xcrun devicectl` supports network deploy after first USB pairing. The cable bottleneck has been logged in three consecutive completion reports as a recurring blocker. It isn't a recurring blocker. It's a configuration step that hasn't been done.

**2. Install build 71 and live with it for 48 hours, with Mei and Cora explicitly in the loop.** Not "show them the panel and ask if they like it." Specific: ask Mei to use Baxter once a day for a real thing she'd want help with; ask Cora to try one thing the panel isn't expected to handle, and one thing it should. Capture their reactions, not your interpretation.

**3. Fix the v2 system prompt completely with v3 — full sweep, not surgical edit.** Find every "Woadhouse" reference. Update every worked example. Update the example for "Tell me a joke" so the model models telling a joke, not refusing. Run `tools/sync-character.sh` after.

**4. Cut the thinking-tick gating (CC's Rec 1) before installing on device.** 30-minute fix. Without it, build 71 will feel slower than build 70 and contaminate the Mei/Cora evaluation.

**5. Reconsider the 0c.19 → 0d.2 ordering.** Wake word is the second non-goal in the brief that has flipped without recorded decision. Writes (0d.2) take voice from 0% capability to actually-useful. Wake word takes the panel from "tap-to-talk" to "always-listening tap-to-talk-replacement" — at the cost of three new external dependencies, more audio session complexity, and the kind of false-positive surface that a deaf household member won't catch but his family will. **Build writes first. Then decide whether wake word is actually wanted.**

**6. Drop the thinking-tick layer.** The acknowledgment phrase already covers LLM TTFT. The tick layers a stall-prefix on a stall-prefix. CC's analysis already shows the tick can produce silence when audio is ready. Remove the bridge entirely and lean on the acknowledgment phrase, which the streaming pipeline already optimizes well. This deletes ~30 audio files, simplifies VoiceCoordinator, and removes the Stage 4 gate. If lived use later shows perceived silence is a problem, add the bridge back with measured evidence.

**7. Rewrite the "silent grace" failure mode to "visible grace."** Failures are always visible in the TranscriptBand, even if no audio plays. "I didn't catch that, sir." is in-character and informative. Pure silence is in-character and useless to a deaf user's family.

**8. Spawn a small "what does the family say" doc in the handoff repo and require it to be updated before any further character iteration.** The discipline is that no v4 system prompt change ships until at least one entry from Mei or Cora exists since the last change. This is the steering signal the loop is missing.

Items 1–4 take less than an afternoon. Items 5–8 are decisions, not work. None of this is in tension with the audit recommendations from DC or CC; it sequences them around the missing-input problem.

---

## 9. What this project should NOT change

Genuine strengths to protect from churn:

- **The Hearth design system.** Mature, internally consistent, doing real work. CC and DC are both right to leave it alone.
- **The 17 principles, with the noted soft spots.** The principles are organizing real decisions. Don't flatten them with new ones (including DC's proposed Principle 18). Tighten what exists rather than adding more.
- **The compile-time PanelConfig pattern.** Resists the permanent gravitational pull toward in-app settings screens.
- **The actor-based HAClient + strict-concurrency-complete.** Hard-won. Don't compromise this when 0d.2 wires writes.
- **The completion-report archive as memory infrastructure.** With the caveat that more reports is not the same as more progress.
- **The VoiceAssistant protocol abstraction.** The Cartesia WebSocket pivot landed cleanly behind it; the SambaNova swap will too.
- **The Cartesia + Benedict voice choice.** I can't audit this audibly, but the rationale (character continuity across LLM swaps) is sound and the SDK abstraction makes the choice cheap to revisit.
- **Local HA URL for wall panels.** Pragmatic and aligned with Principle 15.
- **The "from the future" aesthetic discipline (Principle 11).** It's the project's distinctive thing and it's holding up.

---

## Closing

This project is in better shape than the code-level worry list suggests, and in worse shape than the strategic-audit narrative suggests. The architecture is sound. The principles are organizing real decisions. The voice subsystem is technically impressive.

But the project has been iterating on a voice character for an audience that hasn't been consulted, with a primary user who cannot hear the output, on a cadence that excludes lived testing — and the audit framework (DC + CC + me) cannot fix that from inside the loop. Mei and Cora need to be in the loop as evaluators, not as imagined recipients. Until they are, every iteration is a guess refined against itself.

The pause to audit was the right call. Use the pause to widen the input, not to tighten the existing loop further.

— AUC, end of independent audit
