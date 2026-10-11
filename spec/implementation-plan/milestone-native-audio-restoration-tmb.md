# Native-audio restoration (TMB + guitar) + immersive-audio V2-b scope

Status: 🟢 minimum fix implemented (iOS wiring) · device listening acceptance PENDING · V2-b immersive scope
PROPOSED (STOP for owner before broader V2-b). Date: 2026-10-11. No frozen prompt/contract change. Not pushed.

## 1. Actual TMB root cause (evidence, each layer tested independently — NOT inferred from category)
- **Suspect 1 (missing source audio): RULED OUT.** TMB-type source clips carry audio tracks (verified 2 audio
  tracks on the dir_tmb MOVs). Source ambience exists.
- **Suspect 3 (Edit Planner mutes meaningful sound): RULED OUT.** A reproduced TMB-style hiking EditPlan with
  `native_audio_priority=important` returned `native_audio = keep×4, duck×2` — NO mute of ambience segments.
  The planner is correct GIVEN a correct MusicPlan.
- **Suspect 4 (renderer): RULED OUT.** keep=1.0 + supportive music recedes; the mix preserves keep correctly.
- **ROOT CAUSE = Suspect 2 driven by Suspect 5:** MusicPlan rates TMB native audio `low`/`continuous` because
  it is STARVED of context. Direct test: a hike intent describing "wind, footsteps, conversation, stream"
  → `priority=important, role=supportive` ("ambience is emotionally central"); a thin "a hike" intent →
  `priority=low, role=continuous`. The iOS resolver sent MusicPlan **empty StoryGuide/StoryCoverage** AND the
  picker call sites passed `guideFramingSummary: nil`, so MusicPlan judged from a thin CreativeIntent alone →
  `low` → continuous music drowns wind/footsteps/conversation. Same CLASS as guitar, confirmed independently.

## 2. Guitar wiring-fix status
The approved Part-A fix was **NOT previously implemented** (no commit; resolver still sent empty objects). The
guitar Story only sounded fixed because its CreativeIntent text happened to be rich enough. Now implemented.

## 3. Minimum corrective change (smallest responsible layer = iOS input wiring)
`MusicSelectionResolver.recommend()` now **loads the Story's persisted StoryGuide (story_guide/1) + Coverage
and passes them to the MusicPlan request** instead of empty objects (fetched from SwiftData by storyId, same
pattern as StoryGuideCoordinator; empty object when absent). This gives MusicPlan the evidence to rate native
audio (performance OR meaningful ambience) `important` and keep the soundtrack supportive. Fixes guitar AND
TMB from one root. keep/duck/mute semantics untouched; no global floor; no mute/duck reinterpretation.
NOTE: the two picker call sites still pass `guideFramingSummary: nil` to recommend() — the Guide JSON now
carries the signal regardless, but threading the framing summary too is a cheap follow-up.

## 4. Model/contract implications
NONE. No frozen music_plan/1 prompt change, no contract change, no renderer change, EditPlan/2 untouched. If,
WITH Guide/Coverage passed, MusicPlan still under-rated a clear ambience/performance Story on device, THEN a
versioned music_plan/2 prompt amendment would be proposed + STOP — but evidence says the input gap is the cause.

## 5. Audio regression results (reproduced server-side; DEVICE listening still required)
| Story | MusicPlan priority (with context) | EditPlanner native_audio | Expected mix |
|---|---|---|---|
| Guitar performance | important/supportive | keep on final performance | guitar dominant on payoff ✓ |
| TMB hike (ambience) | important/supportive | keep×4, duck×2 | wind/steps/convo audible ✓ |
| Birthday singing | (expected important) | — | vocals preserved (device) |
| SF music-forward | low/continuous OK | — | music leads ✓ |
| Quiet solitude | (expected selective/important) | — | ambience preserved (device) |
| Explicit music-only | low (user intent) | mute/duck OK | native may drop ✓ |
A schema-valid plan is NOT proof of correct audio — items needing ears are marked (device).

## 6. Physical-iPhone listening checklist (after rebuild)
1. TMB hike: wind, footsteps, and conversation clearly audible; music supports, does not drown.
2. Guitar: final performance dominant.
3. Birthday: singing intelligible.
4. SF montage: music leads (native low acceptable).
5. Solitude: quiet natural ambience preserved.
6. Explicit music-only request: native intentionally dropped (not forced audible).
Report native:soundtrack balance + any masking for each.

## 7. Revised V2-b immersive-audio scope (PROPOSAL — not implemented)
Builds on the V2-b P0 audio-continuity work. Achievable with EXISTING contracts (no new fields):
- **Meaningful environmental-sound preservation** — honored today via the planner's per-segment keep/duck
  once MusicPlan priority is correct (this fix). No new contract.
- **Local RMS/onset event evidence** — already in the V2-b `AVFoundationEvidence` sampler (boundary-only,
  on-device). Extend to flag speech/transient-rich segments. No contract.
- **Audio bridges (J/L)** — already scaffolded in V2-b (deferred to device). No contract.
- **Safeguard against masking speech/performance** — a deterministic renderer rule: when a segment is `keep`
  AND evidence shows strong onsets/voice band, clamp soundtrack gain lower under it. Renderer-only, no contract.
REQUIRES a versioned extension ONLY IF time-varying (within-segment) soundtrack gain is wanted (e.g. duck music
just during a spoken line): that needs a per-segment `soundtrack_gain_envelope` → propose **EditPlan/3**, STOP
for approval. NOT in scope now. NOT building audio separation. NOT building the whole V2-b milestone.

## STOP
Minimum fix implemented (iOS wiring). Device listening acceptance + V2-b immersive scope await owner. Not pushed.
