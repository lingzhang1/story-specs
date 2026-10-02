# Milestone C — Story Director: Implementation Design

**Status:** Approved for implementation design (design only — no real AI built).
**Date:** 2026-10-01
**Component:** Story Director
**Owner:** Director Engine (backend) · consumes CreativeIntent + SceneUnderstanding
**Output contracts:** `StoryConcept`, `StoryPlan` (incl. `StoryBeat[]` + `Shot[]`),
`DirectorDecision`, `DirectorState`/`BeatProgress`
**Complements:** the product/semantics draft `05-story-director.md` (kept as
rationale). Where the two differ, THIS doc + the frozen schema win.

Repo-aware plan for how the Director plugs into the existing system. No real
analysis is implemented — the engine stays mock until the model work is
green-lit.

---

## 0. Contract reconciliation — DONE

Reconciled this milestone (`ai_contracts.py`, JSON Schema, Swift DTOs, contract
tests — 49 passed). Additions, all evidence-minimal:

- **`StoryBeat {id, purpose}`** — minimal planning object; no execution state,
  no speculative fields.
- **`StoryPlan`** gains **`beats[]`** alongside `shots[]`. Planning only — no
  execution progress.
- **`Shot`** gains **`beat_id`** + **`purpose`** (internal semantic reason);
  keeps canonical `framing`/`camera_motion`/`subject_action`/`instruction`/
  `target_duration_seconds` (no `camera_behavior`/`subject_behavior` dupes).
- **`StoryConcept`** keeps `summary` AS the creative thesis (no `thesis` dupe).
- **`DirectorDecision {action, shot?, reason?}`** — new; `action ∈
  {capture_shot, create_now, rescan_scene, retry_shot}`. One strongest action.
- **`BeatProgress {beat_id, status, supporting_asset_ids?}`**, `status ∈
  {unfulfilled, partial, fulfilled}` — Director state, categorical, no numeric %.
- **`DirectorState {current_beat_id?, beat_progress[], last_decision?}`** — new.
- **`CaptureSession`** gains **`director_state`** (owns the loop state).
- **`ValidationResult`** stays shot-level; gained optional `shot_id` +
  `capture_asset_id` references (NO beat-fulfillment score).

Boundary (frozen): `StoryPlan` = intent; `DirectorState` = what happened;
`ValidationResult` = "did we get the requested Shot?"; `BeatProgress` = "did it
fulfill the Beat?" (Director's call, not the Validator's).

---

## 1. Director initialize flow

```
INITIALIZE
  CreativeIntent + SceneUnderstanding + CaptureConstraints
        ↓  (one Model Gateway structured call)
  StoryConcept                 (title + summary/thesis)
  StoryPlan { beats[], shots[] }   (concept → beats → shots; every shot.beat_id set)
  initial DirectorState { current_beat_id, beat_progress=[unfulfilled…] }
  initial DirectorDecision { action: capture_shot, shot: <first beat's shot> }
```
Director plans the Story **globally** (concept + beats + the shots it currently
intends) and returns the **first** next action. Whether all shots are
materialized up front or incrementally is an OPEN experiment (§11) — the contract
supports either (shots may be added to `StoryPlan.shots` as the loop proceeds).

## 2. Director next-action loop

```
NEXT
  CaptureSession.director_state (prior) + latest capture + ValidationResult
        ↓  (one Model Gateway structured call)
  Director judges beat fulfillment from: the beat, the requested Shot, the
  ValidationResult, and captured assets  → updates BeatProgress
        ↓
  updated DirectorState + next DirectorDecision (one of):
     capture_shot  — next highest-value unfulfilled beat's shot
     retry_shot    — same beat, clip usable-but-weak or a retake issue
     rescan_scene  — scene context stale/insufficient
     create_now    — enough material; stop asking for shots
```
"Plan globally, direct incrementally" (05 §5). The loop ENDS when Director
chooses `create_now` (minimum-sufficient capture, 05 §8) — not when a fixed shot
list is exhausted. Initialize and NEXT are **logically distinct** (05 §9) but may
share one model/endpoint/prompt — not pre-separated infrastructure.

## 3. State ownership

`CaptureSession` is the aggregate (05 §8). It already holds `creative_intent`,
`scene_understanding`, `selected_concept`, `story_plan`; it now also holds
`director_state`. The **LLM is stateless**; the domain state is passed in and
out each call. On iOS, `CaptureSession`/`DirectorState` are transient to the
capture context and need not be fully persisted as SwiftData (ephemeral, like
SceneUnderstanding) — persist only what a resumed capture needs. Keep the plan
(intent) and the director_state (progress) separate in storage as in the contract.

## 4. Model Gateway integration

Director uses the **generic** structured-request gateway (shared with Creative
Intent / Scene Understanding): `task="story_director.initialize"` /
`"story_director.next"`, `schema=<StoryPlan+DirectorDecision envelope>`,
`context={intent, scene, prior director_state, latest validation}`, `images=[]`
by default (raw-frame access is an experiment, §11). No provider SDK in Director
domain code; model is config, keyed by task (03 §15, §34).

## 5. Structured-output strategy

Director returns a single structured envelope (concept + plan + decision for
initialize; updated state + decision for next), validated against the frozen
schemas (`extra="forbid"`). Bounded gateway repair/retry; typed
`StructuredOutputError` on persistent failure → §8 behavior. Internally the model
may consider several candidates (05 §9) but MUST emit ONE decision — never a menu.
A post-validator enforces: every `shot.beat_id` references a real beat; no
edit/scene/safety-certification language leaks into shots.

## 6. Feasibility & safety constraints (05 §11, §21)

Before recommending a shot, Director applies a feasibility check against
SceneUnderstanding: subject exists? `operator_space` supports it? physically
reasonable with a phone? simple enough? Physical feasibility OUTRANKS cleverness.
**Safety is a Director constraint, not a Scene conclusion**: Director operates
under conservative capture rules regardless of model confidence — never
recommends walking backward near roads/edges, entering traffic, climbing unstable
structures, or interfering with strangers. These rules live in the prompt + a
deterministic guard, not in SceneUnderstanding.

## 7. Prompt architecture

Versioned/external: `prompts/story_director/v1/{system.md, examples.json,
schema.json}` + `current`. One prompt may serve initialize + next (open question
§11); the prompt demands scene-specific concepts, concept→beats→shots coherence,
one strongest decision, minimum-sufficient capture, and the two-layer Shot
(purpose vs instruction). It prohibits generic filmmaking templates
("wide → medium → close-up"), music selection, and persistent-Story orchestration.

## 8. Retry / failure behavior

Mirrors the other engines (05 §19 loop, no silent fabrication):
- model/network/structured-output failure → preserve the CaptureSession +
  DirectorState, surface a recoverable retry state; do NOT fabricate a plan.
- `retry_shot` is a first-class *successful* decision (clip usable-but-weak), not
  an error path.
- `MockDirectorEngine` stays dev/tests only — never a production plan.

## 9. Observability

Per call, outside the domain objects: `operation`
(initialize|next), `model`, `provider`, `prompt_version`, `latency_ms`,
`attempts`, `structured_output_ok`, `decision_action`, `beat_count`,
`shots_requested_so_far`, `usage`/cost, `correlation_id`. These feed the
min-capture and utility evals (§10-12). Reuses the 03 §37a log shape.

## 10. Scene-specificity evaluation (the top criterion, 05 §10)

Fixtures: same CreativeIntent (e.g. "cinematic and energetic") across materially
different scenes — office, Times Square, TMB trail, birthday dinner, hotel room,
beach. **Pass = the plans differ and exploit scene-unique properties**; fail = the
generic "wide → medium action → close-up" template. This is the single most
important Director quality gate.

## 11. Minimum-capture evaluation (05 §8)

Measure shots requested vs. Story quality: does Director stop (`create_now`) when
two clips suffice instead of demanding five? Does it reuse existing Moments rather
than request redundant captures? Reward quality/effort, not maximal footage.

## 12. Director utility evaluation (05 §3, downstream)

Compare Director output quality across inputs: intent-only vs intent+basic scene
labels vs intent+full SceneUnderstanding (ties to the Scene Understanding Layer-2
eval). Also validate that `beat_id` tagging gives the future Edit Planner usable
"why this footage exists" signal.

## 13. Interaction with Validation (05 §6)

`ValidationResult` (shot-level: usable? recommendation accept/retake? issues[])
feeds the NEXT call. Director maps it onto `BeatProgress`: a usable clip may still
leave a beat `partial` (e.g. scale not strong enough) → Director may `retry_shot`
or accept and move on. The Validator never sets beat status; Director does.

## 14. Interaction with the future Edit Planner

`StoryPlan.beats` + `Shot.beat_id` + `BeatProgress.supporting_asset_ids` give the
Edit Planner the narrative role of each clip ("establishes scale", "progression",
"release") instead of opaque clip ids — a direct downstream payoff of beats
(05 §3). Director does NOT produce an EditPlan; it stops at
`DirectorDecision=create_now`, which hands off to Create My Story.

## 15. Director principles (frozen, 05 §20)

1. Concept → Beats → Shots. 2. Plan globally, direct incrementally. 3. One strong
recommendation at a time. 4. Minimum sufficient capture. 5. Every Shot serves a
Beat. 6. Every Beat serves the StoryConcept. 7. Scene specificity mandatory.
8. Feasibility > cleverness. 9. Shot semantics ≠ user instruction. 10. Validation
feeds the loop. 11. Director may decide enough exists. 12. Director ≠ persistent-
Story orchestration. 13. No music selection in v1. 14. Raw-frame access is an
experiment. 15. No fields without demonstrated value.

## 16. Open experiments — KEEP MEASURABLE (do not silently resolve, 05 §11)

- Whether Director also receives raw scene frames (not just SceneUnderstanding).
- Optimal number of Beats.
- Whether all planned Beats need explicit Shots up front.
- Whether Director plans all Shots or materializes them incrementally.
- Whether `BeatProgress.supporting_asset_ids` is needed.
- Exact model/provider.
- Whether initialize/next use one prompt or two.
- How much persistent-Story context Director eventually receives.

## 17. Definition of Done (implementation)

- Director initialize produces a scene-specific `StoryConcept` + `StoryPlan`
  (beats + shots) + first `DirectorDecision` via a real model through the gateway;
- the next-action loop updates `DirectorState`/`BeatProgress` from captures +
  `ValidationResult` and returns one `DirectorDecision`;
- output validates against the frozen schemas; failures retry without losing
  CaptureSession/DirectorState;
- scene-specificity, minimum-capture, and utility evals pass their bars;
- no music selection, no persistent-Story orchestration, no fabricated plans.

Then Story Director becomes the live core of DIRECT. **No real AI is implemented
by this document.**
