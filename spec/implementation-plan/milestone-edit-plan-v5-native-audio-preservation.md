# Native-audio preservation v5 + Customize Your Story — staged plan

Status: 🟡 STAGE 1 done (edit_plan/5 authored + v4-vs-v5 regression). **AT THE v5 FREEZE GATE — awaiting owner.**
Stages 3–7 (persistence, wiring, UX screen, renderer verify, device accept) NOT started. No frozen v4 edit.
Not pushed. Date 2026-10-11.

## 1. v5 prompt delta (the ONLY change vs frozen v4)
New file `prompts/edit_plan/5/system.md` = v4 VERBATIM except the **NATIVE AUDIO** clause. No schema change
(segments still carry keep|duck|mute). Selected by env `STORY_EDIT_PLAN_VERSION=5`; default stays `4`
(production untouched); unknown value falls back to `4`. `packages/edit_planner/__init__.py` resolves the
version/path from that env.

v4 (frozen): "If a Moment's own sound is the emotional payoff (a real guitar performance, birthday singing,
meaningful dialogue), native_audio=keep. **Otherwise duck or mute.**" → a fixed category list; everything else
reduced. This is why the hike (no "performance"/"dialogue") went duck×12/mute×6 → music-only.

v5 (the 7 owner principles):
- **DEFAULT TO keep.** Original captured sound is first-class material: environmental/atmosphere (wind, water,
  footsteps, street, crowd), human sound (performance, singing, laughter, meaningful conversation), texture.
- **NO fixed category list** — judge each segment's own audio on whether it serves THIS Story; do not reduce
  merely because it is "not dialogue or performance".
- **Reduce only for a specific EDITORIAL reason** (unusable audio; it masks something more important in the
  same stretch; the stretch is deliberately music-forward; the user asked). duck = present-but-under; mute =
  genuinely should not be heard.
- **Respect explicit preference** — music-led/only intent MAY duck/mute broadly (valid result); immersion/
  ambience biases hard to keep; priority important/selective → keep, low → broader duck/mute permitted.
- **Do NOT force every sound audible** — a stretch may legitimately be music-only; keep is the starting point,
  each duck/mute is deliberate.

## 2. Regression — v4 vs v5 (real Opus planner, synthetic inputs; local)
| Scenario | v4 native_audio | v5 native_audio | Read |
|---|---|---|---|
| TMB hike (immersive) | keep×6 | keep×6 | both keep* |
| Guitar performance | keep×3 | keep×3 | preserved ✓ |
| Birthday singing | duck×2 keep×2 | **keep×4** | v5 stops ducking the singing/laughter ✓ |
| SF montage (music-fwd) | duck×1 mute×5 | duck×2 mute×4 | music-forward honored by BOTH ✓ |
| Solitude quiet | keep×3 | keep×3 | ambience preserved ✓ |
| Explicit music-only | mute×4 | **mute×4** | v5 STILL allows intentional music-only ✓ |

GATE CHECKS: (a) v5 lifts meaningful ambience/human sound to keep where v4 ducked it (Birthday). (b) v5 does
NOT break intentional music-only (Explicit mute×4) or music-forward montage (SF mostly mute/duck). (c) feasible
for all; schema unchanged; renderer-compatible (same keep/duck/mute values the mixer already consumes).
LIMITATION (honest): with rich synthetic summaries even v4 kept TMB — the DEVICE TMB sent THINNER moment
summaries/events, which is why it got duck×12/mute×6. So this harness validates v5's DIRECTION and the
music-only safeguards; it does NOT reproduce the device duck/mute. True proof needs a DEVICE re-render on v5.

## 3–7. Remaining stages (NOT started — owner sequence, each independently testable)
3. **Sound-preference data model (Part C):** add `intentSoundPreference: String?` (enum immersive|balanced|
   music_led; nil = infer) to StoryModel — the SMALLEST change. Threads into MusicPlan.audio_strategy
   (role/priority) and is passed to the planner as an explicit-preference input (v5 already honors it). NOT a
   frozen-contract change if carried as an existing CreativeIntent field; if the music_plan/edit_plan request
   DTOs cannot carry it, propose a minor additive request-field extension (no response/schema change). Must NOT
   be silently overridden on regenerate — persist on the Story, re-read each run.
4. **Input wiring:** resolver/coordinator pass the preference into /v1/music-plan and /v1/edit-plan.
5. **Customize Your Story screen (Part B):** replace the duration sheet with Length (Highlights/Full/Custom via
   Duration Intelligence, no hardcoded durations, whole-row clickable) + Sound (Immersive/Balanced/Music-led,
   default Balanced unless CreativeIntent is clear). Figma visual language; no mixer/sliders.
6. **Renderer verify:** confirm keep/duck/mute + role map to the intended audibility; the duck-under-continuous
   observation stays flagged, not changed (owner: no global duck/role reinterpretation).
7. **Device acceptance:** the Part E checklist on real footage, comparing rendered AUDIO not JSON.

## 6. Contract implications
edit_plan/5: prompt-only, additive, env-gated — EditPlan/2 schema UNCHANGED. Sound preference (stage 3): aim to
carry inside CreativeIntent (no frozen change); only if impossible, a minimal additive request field. No frozen
EditPlan/2 or music_plan/1 semantic change. mute is never globally turned into duck.

## STOP — v5 FREEZE GATE
Stage 1 complete and passing. Per Part D, STOP here for owner freeze approval before stages 3–7 and before any
production integration. On approval: freeze edit_plan/5 (flip the default or deploy with the env set) and
proceed to stage 3. Not pushed.
