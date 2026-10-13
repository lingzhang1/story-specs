# Milestone — Multimodal Creative Editing: Visual Evidence & Director-Level Reasoning (Experiment)

Status: 🟡 Phases 0–2 done + prototype designed; **STOPPED at the $20 A/B/C spend gate and the architecture
gate** awaiting owner go. No production change. No frozen prompt/contract touched. Nothing pushed.

Owner question: should the Edit Planner also SEE actual video frames and reason visually, instead of only
reading distilled AU text? Objective: better artistic judgment, shot selection, source-range precision,
visual continuity, pacing. Explicit caution: more frames / more tokens are NOT assumed to improve quality.

---

## Phase 0 — Current implementation, verified against code (not prompt descriptions)

Pipeline today (every claim read from source this turn):

| Stage | Prompt / file | Model | Sees pixels? |
|---|---|---|---|
| Asset Understanding (AU) | `prompts/asset_understanding/v1/system.md`, schema `moment_understanding/1` | image-capable AU model (deps `get_asset_understanding_engine`) | **YES — 24 frames** |
| Moment frame extraction | `apps/ios/.../AssetUnderstanding/MomentFrameExtractor.swift` (`au_frames/1`) | on-device AVFoundation | n/a |
| Edit Planner | `prompts/edit_plan/5.2/system.md` (dev default 5.1), engine `packages/edit_planner/__init__.py` | Opus 4.6 (Sonnet 4.6 fallback) | **NO — text only** |
| EditPlan output | contract `edit_plan/2` (`edit-plan-v2.schema.json`) | — | — |
| Deterministic validator | `validate_plan()` in `packages/edit_planner/__init__.py` | pure, no model | — |
| Creative Refinement V2-b | `apps/ios/.../Render/CreativeRefinementV2b.swift` | pure/on-device (motion+audio) | partial (tiny luma) |
| MusicPlan / MusicAnalysis | `prompts/music_plan/v1` (`music_plan/1`); `music_analysis/1` | Opus 4.6 / provider | — |
| Media Engine | `apps/ios/.../Render/MediaEngineV1.swift` | on-device AVFoundation | — |

Datasets on disk (raw video only — **no stored AU/moment manifests**):
`evals/scenarios/girl_with_dress` (36 clips: 34 .mov/.MOV + 2 .HEIC), `evals/scenarios/hiking` (~40 clips),
`evals/scenarios/guitar_practice_001` (7 clips + `editorial_reference.APPROVED.json`). Solitude: not a
scenario dir; prior runs used a 3-clip solitude payload. Reusable visual infra already exists:
`evals/asset_understanding/make_contact_sheets.py` (+ `ground_truth.json`, `subset.json`).

## Phase 1 — Evidence & information-loss audit (the core finding)

Information flow, end to end:

1. **Sampled frames contain:** 24 ordered, evenly-spaced frames across the whole clip, longest edge 768px,
   JPEG q≈0.6, timestamped (ms). Evenly spaced → a short, dense action between two samples can fall in the
   gap. No higher density around motion. (`MomentFrameExtractor.swift`, `frameCount = 24`.)
2. **AU recognizes:** from those 24 frames the AU model produces `observed` = summary, subjects[], ordered
   events[] (start/end + factual description), temporal_order_note, and **useful_segments[]** (start/end +
   why_useful); plus `story_context` (story_utility, requested shot/beat contribution). It is told to reason
   ACROSS frames as one Moment and to prefer saying less over inventing.
3. **AU persists:** exactly the `moment_understanding/1` record — **text only**. Frames are produced
   on-device for the one AU request and **discarded**; the original video never leaves the device; no frame
   store exists. (`AssetUnderstandingService.swift` sends multipart frames; nothing persists them.)
4. **Edit Planner receives (per Moment):** `{ id, duration_seconds, useful_segments: [[a,b],…],
   summary }` where `summary` is the **folded AU evidence string** (summary · subjects · events · temporal
   note, de-duped, capped at 600 chars — the "Arm B AU-evidence forwarding" in `RenderSupport.swift`
   `understandingSummary`). Plus CreativeIntent, StoryGuide, StoryCoverage, MusicPlan, optional
   MusicAnalysis, sound_preference, editorial_mode. **Zero images. Zero per-event timestamps** — the planner
   has useful_segments for timing but not the AU event times (`auActionTimestamps` is `[:]` today).
5. **Information lost between AU and the planner:**
   - **All pixels.** The planner never sees composition, framing, color, light, who-is-looking-where, or
     the actual look of an outfit/location — only AU's prose about them.
   - **Everything AU saw but didn't write.** AU's 600-char folded string is a lossy compression of a lossy
     compression (24 frames → prose → capped prose). Distinct outfits/accessories AU under-described never
     reach the planner; this is the measured mechanism behind the earlier 16→22 girl_with_dress gain from
     merely forwarding MORE of the SAME text.
   - **Event timestamps.** AU events carry times; the planner gets none, so it cannot align a cut to a
     specific action beat except via the coarser useful_segments.
   - **Sub-frame-gap actions.** A fast action between two of the 24 evenly-spaced frames may never be seen
     by AU at all → cannot be in useful_segments → unselectable.
6. **Are useful_segments precise enough for short-form editing?** Partially. They are coherent candidate
   REGIONS ("complete action of X"), not frame-precise in/out points. The renderer's Creative Refinement
   V2-b exists precisely because the plan's cut points need nudging — but it refines from **tiny 16×16 luma
   motion + audio RMS**, not from understanding, and it is **OFF by default** (`StoryCreativeRefinement.enabled
   = false`, not device-accepted). So today nothing produces understanding-driven frame-precise boundaries.
7. **Can the renderer execute the creative choices under consideration?** The Media Engine executes: source
   in/out, speed, cut/crossfade/fade/dip transitions, per-segment native-audio keep/duck/mute, music-beat
   snapping (`snappedCut`), crop. It does **not** execute: match cuts, speed ramps, color, split-screen,
   reframing-by-content (explicitly out of V2-b scope). So "see frames and pick a tighter in-point / a more
   distinct shot / a better payoff hold" is executable TODAY; "match-cut on action" or "reframe to subject"
   is not — any visual-evidence experiment must stay inside the executable set.

**Failure examples from real footage (observed this cycle, grounded in the dev logs + prior device runs):**
- girl_with_dress collapsing to 7–15 clips was NOT the creative prompt — it was (a) Opus day-cap → Sonnet
  fallback and (b) short/unstable target duration (40s vs 72s on near-identical input). Separately, the
  "full story ≈ highlights" convergence is a breadth problem the planner can only solve with the evidence
  it has — thin AU summaries give it little to justify distinct selection. This is the clearest motivation
  for richer evidence: the lever that moved selection (16→22) was MORE evidence, text so far.
- Even-sampling gap risk: a 375s outlier clip (IMG_9162) sampled at 24 frames = 1 frame / ~15.6s; any
  discrete action shorter than that window is invisible to AU. Density, not modality, is the issue there.

**Phase-1 verdict:** the single biggest information loss is **AU→prose→cap**, not the absence of pixels per
se. The experiment must separate two distinct hypotheses: **(H1) the planner needs richer/less-lossy
evidence** (could be more text, or images) vs **(H2) the planner specifically needs PIXELS to make a
better artistic judgment than any text can carry** (composition, look, continuity). A/B/C is designed to
tell these apart rather than assume H2.

## Phase 2 — Visual-evidence prototype (experiment-only; designed, not yet run)

Representation (reusing `make_contact_sheets.py`, extended — review infra, not a product path):
- **Tier-1 (every Moment):** a small timestamped **contact sheet** — N representative frames tiled with
  their source timestamps burned in, N scaled to duration (e.g. 4–9 frames, denser for long/multi-event
  clips), longest edge ≤768, JPEG. Stable Moment IDs + timestamps preserved so a visual observation maps
  back to a source range the planner/validator can use.
- **Tier-2 (targeted, only where justified):** higher-density frames around a candidate action/highlight
  the Tier-1 sheet or AU events flag — added ONLY for Moments where Tier-1 shows a precise-boundary gap.
- Guardrails honored: no fabricated observations; **no permanent frame store**; **no upload of full
  original videos** (only derived, bounded JPEG frames); temporary bounded image inputs only.

What the prototype evaluates visually (named, so each has a consumer): clothing/accessories, locations,
subject + camera movement, composition, visually distinctive actions, candidate highlights, source-range
boundaries. Each maps to an existing EditPlan field (selection, order, source in/out, narrative_role,
native_audio, music_alignment) — **no new contract field is introduced by the experiment.**

## Phase 3 — Controlled A/B/C (DESIGNED — gated at $20 model spend)

Hold constant: CreativeIntent, StoryGuide/Coverage, editorial_mode, target duration, MusicPlan/MusicAnalysis,
EditPlan/2 schema, deterministic validation, model config. Vary ONLY the evidence given to the planner.

- **Arm A — text-only (control):** today's path, v5.2, current rich-AU forwarding. The planner call is the
  production shape.
- **Arm B — text + visual contact sheets:** same model/prompt body; the user message additionally carries
  the Tier-1 contact-sheet images per Moment. Because the production planner call is text-only, Arm B
  REQUIRES an **experimental multimodal prompt wrapper** (a Converse request that attaches image blocks
  alongside the existing JSON). This wrapper is a documented, experiment-only delta — **it is NOT a model
  change and must not be conflated with one.** The system prompt body stays byte-identical to v5.2; only the
  transport adds image parts + one line telling the model the sheets are timestamped evidence for the same
  Moments.
- **Arm C — B + targeted higher-density frames** around candidate highlights, run **only if B shows a
  meaningful source-range-precision gap** vs A. (Conditional by design — avoids spending on C if B already
  closes the gap or shows no benefit.)

Fixtures: girl_with_dress, hiking, guitar_practice_001, + solitude payload. Spend gate: **$20 initial**;
stop and report if hit before all arms complete. Each arm's exact request (model id, max_tokens,
temperature 0, prompt_version, image count/bytes) is logged so cost/latency is attributable.

## Phase 4 — Director-level quality evaluation (DESIGNED)

Scored SEPARATELY per arm, NOT by clip count: narrative coherence, visual variety, shot selection,
source-range quality, pacing, continuity, essential-action preservation, music relationship, native-audio
decisions, executability. Reference: owner's human-edited girl_with_dress edit (if accessible) +
`guitar_practice_001/editorial_reference.APPROVED.json`. Invariants to protect: guitar's long continuous
performance preserved; solitude NOT forced into short cuts.

## Phase 5 — Candidate planning & critique (DESIGNED — not implemented)

Compare the potential value of: (1) one direct EditPlan; (2) two alternative creative candidates;
(3) a critique/revision pass. Decide whether extra model calls buy enough quality to justify cost+latency.
NOT assumed that more calls / a bigger model help. This is a paper design pending A/B/C evidence, because
whether a critique pass helps depends on whether the planner's errors are evidence-limited (fix with
evidence, Phase 3) or reasoning-limited (fix with critique, Phase 5).

## Phase 6 — Architecture recommendation (minimum evidence-supported design)

Decision tree, resolved AFTER A/B/C — current lean, to be confirmed by evidence:
1. **If richer AU text closes most of the gap** (likely, given the 16→22 text-only result): expand what AU
   forwards (raise the 600-char cap, thread AU event timestamps into `auActionTimestamps`) — cheapest,
   on-device-friendly, no new model pass, no new contract. **Start here.**
2. **Targeted visual contact sheets to the planner** (if B beats A on source-range/visual-variety by a
   margin that justifies image tokens): backend planner call, experiment wrapper graduated to a real
   multimodal request; frames derived on-device, bounded, never persisted.
3. **Separate visual highlight refinement** (device, understanding-driven boundaries): graduate Creative
   Refinement V2-b from luma/audio to AU-evidence-driven, flip it on after device acceptance.
4. **Bounded candidate + critique stage** only if Phase 5 shows reasoning-limited errors.
5. **No additional model pass** if A already matches the reference within tolerance.

On-device vs backend / privacy / cost: AU + frame extraction stay on-device (originals never leave the
device). Any image-to-planner path uploads only derived, bounded, transient JPEGs — no full video, no frame
store, caching only within one Story's planning request. **Every proposed new evidence field must name its
consumer; none is added until a concrete gap is demonstrated.** Frozen contracts (`moment_understanding/1`,
`edit_plan/2`, `music_analysis/1`) are preserved unless A/B/C demonstrates a field they cannot express.

## Final report (phases 1–10 condensed)

1. **Information flow:** on-device 24 frames → AU text (`moment_understanding/1`) → **text-only** planner
   (`edit_plan/2`). Pixels seen only by AU, then discarded.
2. **Visual prototype:** Tier-1 contact sheets per Moment + optional Tier-2 density; reuses
   `make_contact_sheets.py`; no frame store, no full-video upload.
3. **A/B/C:** designed, held-constant list fixed, gated at $20 — **not yet run.**
4. **Source-range accuracy:** today bounded by useful_segments (regions, not frame-precise); refinement is
   off + luma-only. The prototype targets exactly this.
5. **Editorial-quality comparison:** scored per Phase-4 axes, clip count NOT primary — pending A/B/C.
6. **Cross-domain regressions:** guitar continuous-hold and solitude-calm are the guard rails — pending.
7. **Model cost/latency:** per-arm request logging designed; the $20 gate bounds spend.
8. **Architecture recommendation:** cheapest-first ladder (richer AU text → targeted sheets → device visual
   refinement → candidate/critique → nothing), resolved by A/B/C evidence.
9. **Minimal implementation plan:** step 1 (richer AU text forwarding + event timestamps) is a no-new-model,
   no-contract change and is the recommended first move regardless of A/B/C outcome.
10. **Owner freeze decisions needed:** (a) authorize the $20 A/B/C run incl. the experiment-only multimodal
    wrapper on the DEV backend; (b) confirm uploading bounded derived frames to the dev planner for the
    experiment is acceptable; (c) whether to ship step-1 (richer AU text) immediately as it needs no gate.

**STOP for owner review before any production change. No AU or Edit Planner prompt frozen. No NL editing or
advanced effects. Unrelated changes preserved. Nothing pushed.**

---

# ADDENDUM — A0/A1/B experiment EXECUTED (owner-approved $20 gate), 2026-10-12

Runner: `evals/multimodal_edit/run_experiment.py` (experiment-only; frozen v5.2 body verbatim across arms;
EditPlan/2 schema + deterministic `validate_plan` imported from `packages/edit_planner`; model, temperature 0,
max_tokens identical across arms; only the per-Moment EVIDENCE differs). Multimodal uses the gateway's
EXISTING `StructuredRequest.images` surface — **no model change, no new wrapper, no prompt/contract edit.**
Fresh AU is REAL `asset_understanding/v1` output (Sonnet 4.6, 24 frames au_frames/1), cached, cost-tracked,
labeled real. Result JSON: `evals/results/multimodal_edit_experiment.json` (gitignored; on disk).

**Arms as run:** A0 = production 600-char folded summary + useful_segments. A1 = richer AU text (fuller
subjects, events WITH timestamps, temporal note, NOT 600-capped; no fabrication). B = A1 + ONE timestamped
Tier-1 contact-sheet JPEG per Moment (4–9 duration-scaled frames tiled + labeled). **C = NOT RUN — gated on
owner review** (see below). **Total spend $6.95 / 52 calls — well under $20.** Every arm produced a
deterministically-VALID EditPlan/2.

## Results

| Dataset | A0 | A1 | B |
|---|---|---|---|
| **girl_with_dress** (36) | 21 seg / 21 distinct / med 5.0s | 22 / 18 / 4.55s | 17 / 17 / 6.1s |
| **hiking** (40) | 14 / 14 / 7.5s | 12 / 12 / 8.75s | 13 / 13 / 8.0s |
| **guitar** (6, regression) | 6 / 6 / 7.0s | 6 / 6 / 7.2s | 6 / 6 / 8.4s |

Source-range precision (the real multimodal signal): on **shared** moments, B shifts source in/out vs A1 by
**median 3.0s (girl), 4.8s (hiking)** — B is making different, visually-informed boundary choices, not just
reshuffling selection. B also re-selected footage (girl A1∩B = 12; it dropped some A1 picks and added others)
and consolidated toward fewer, longer holds.

## Findings (clip count is NOT the metric)

1. **Information loss is real and measured.** A0's 600-char cap truncates mid-sentence on rich clips — e.g.
   IMG_2359's A0 string hit the cap at "…contemplative po" and **dropped the entire subjects list** (green
   tassel earrings, black sunglasses, floral dress — the exact visual-distinctness cues) and the event
   timeline. A1 keeps all of it + event timestamps. This is the lever behind the earlier 16→22 text-only gain.
2. **A1 (richer text) is a cheap, real win on dense footage** and essentially free (~$0.02/arm, no images).
3. **B (visual) changes JUDGMENT, not quantity.** It moved source ranges by seconds and re-weighted selection
   toward distinct looks + longer holds — consistent with "see the frames, pick the better window." Whether
   those moves are *better* needs the human/visual review the owner's criteria require (not model confidence).
4. **Regression guards held.** Guitar 6/6 all arms, native keep, the final-song preserved as ONE continuous
   hold. **Watch-item:** B started the final-song hold at 15s vs A0/A1's 3s (36.3s → 24.3s) — not fragmented
   (continuity intact), plausibly trimming tuning lead-in, but needs the human ear to call better-or-worse.
   Solitude: not run (no dedicated dir); hiking's calm clips are the nearest proxy and B did not fast-cut.
5. **Scaling wall discovered:** the naive "all Moments' frames as separate image blocks" overflowed Converse
   ("Input is too long") at 36 moments. ONE tiled contact-sheet per Moment fixed it (36 moments → 36 images,
   ~5MB). Any production visual path MUST tile, not stream raw frames.

## Cost / latency / privacy

- Cost: AU (fixed setup, cached) dominates (~$5–8 for 76+ clips); the planner arms themselves are cheap
  (A0/A1 ~$0.1–0.2 each; B ~$0.4 incl. ~5MB images). Latency: A0/A1 ~21–35s; B ~27–34s (images add little).
- Privacy/transport: only bounded derived JPEGs sent to the DEV planner; no full video; no frame store; scratch
  session-reclaimed; no image payloads logged (counts + bytes only). Retention = in-request only.

## Decision — Arm C gating

C's trigger ("B shows a meaningful remaining source-range-precision gap") is **arguably met** — B does move
boundaries — BUT whether B's boundaries are already good enough (making C unnecessary) vs still imprecise
(justifying C) **can only be judged by the human/visual review the owner mandated.** Running C now would spend
more without that ground truth. Recommendation: **owner reviews the sample contact sheets + the A1→B boundary
shifts; THEN decide C.**

## Architecture recommendation (minimum evidence-supported)

Resolving the owner's decision tree against the data:
- **A1 substantially helps and is nearly free → ship richer AU text + event timestamps first** (step 1). This
  is the recommended minimum: no new model pass, no image cost, no contract change (fields already exist;
  raise/remove the 600-cap in `understandingSummary`, thread AU event times into `auActionTimestamps`).
- **B adds visual JUDGMENT** but at ~5MB/request and a tiling requirement, and its *quality* edge is unproven
  without human review → **selective multimodal planning is a candidate, NOT yet recommended** — gate on the
  human review + a device render.
- **C (targeted density)** only if human review says B's precise boundaries still miss.
- The guitar final-song trim shows some errors may be **reasoning/criteria**, not evidence — so a
  candidate/critique pass (Phase 5) stays on the table independent of visual evidence.

**Production implementation sequence (proposed, NOT executed, owner-gated):** (1) richer AU text forwarding +
event timestamps [no gate needed beyond owner go; no contract change]; (2) IF human review favors B: a
selective, tiled, bounded contact-sheet path for the backend planner, device-gated; (3) C/critique only if a
measured gap remains.

**STOPPED at the architecture/freeze decision. No production change, no new frozen prompt, nothing pushed,
no unrelated work. Arm C not run pending owner review.**
