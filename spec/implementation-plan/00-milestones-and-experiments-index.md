# Story — Milestones & Experiments Index

Single consolidated summary of every milestone and experiment, newest phase last. Each row links to its
authoritative record under `spec/implementation-plan/`. Legend: ✅ closed/device-accepted · 🟢 built &
deployed/verified · 🟡 design/experiment done, awaiting go or at a gate · 🔵 design/reconciliation only.

> Maintenance: when a milestone opens or changes state, add/update its row here AND keep its own doc the
> source of truth. This index is a map, not a replacement for the records.

Repos: app `/Users/lingzhang/Workplace/story` (GitHub lingzhang1/story, `master`); specs this repo.
Dev backend: App Runner `story-development-api`, acct 989088054552 us-east-1,
https://ydmju5wdps.us-east-1.awsapprunner.com. Experiment result JSONs live under `evals/results/`
(gitignored — on disk, not committed). Product source of truth: `spec/production/story-production.md`;
build scope: `spec/implementation-plan/mvp-build-plan.md`; frozen contracts:
`spec/implementation-plan/02-core-story-schema.md`.

---

## Foundations & design specs (01–13)

| Doc | What | Status |
|---|---|---|
| [01 project-foundation](./01-project-foundation.md) | Project foundation | spec |
| [02 core-story-schema](./02-core-story-schema.md) | **Authoritative shared contracts** | frozen |
| [03 system-architecture](./03-system-architecture.md) | System architecture | spec |
| [04 aws-infrastructure](./04-aws-infrastructure.md) · [creative-intent](./04-creative-intent.md) · [scene-understanding](./04-scene-understanding.md) | Infra + CI/scene specs | spec |
| [05 story-director](./05-story-director.md) · [06 asset-understanding](./06-asset-understanding.md) · [07 camera-capture](./07-camera-capture.md) | Director / AU / capture specs | spec |
| [08 shot-validation](./08-shot-validation.md) · [09 edit-planner](./09-edit-planner.md) · [10 media-engine](./10-media-engine.md) | Validation / planner / engine specs | spec |
| [11 direct-flow](./11-direct-flow.md) · [12 edit-flow](./12-edit-flow.md) · [13 evaluation](./13-evaluation.md) | Flow + evaluation specs | spec |
| [mvp-build-plan](./mvp-build-plan.md) · [ui-capability-matrix](./ui-capability-matrix.md) | MVP scope + UI matrix | spec |

## Milestone A–B — walking skeleton + schema freeze

| Doc | What | Status |
|---|---|---|
| (A — in briefing/mvp-build-plan) | DIRECT+EDIT skeleton, mocked AI, on device | ✅ device-verified |
| [milestone-b-reconciliation](./milestone-b-reconciliation.md) | Schema reconciliation; freeze 4 model layers | ✅ complete |

## Milestone C — real AI DIRECT (Creative Intent · Scene · Director · Shot Validation · UX)

| Doc | What | Status |
|---|---|---|
| [c-creative-intent-design](./milestone-c-creative-intent-design.md) · [c1 eval-report](./milestone-c1-creative-intent-eval-report.md) | Creative Intent design + eval | 🟢 built |
| [c-scene-understanding-design](./milestone-c-scene-understanding-design.md) · [c2 report](./milestone-c2-scene-understanding-report.md) | Scene Understanding | 🟢 deployed/verified |
| [c-story-director-design](./milestone-c-story-director-design.md) · [c3a offline-eval](./milestone-c3a-director-offline-eval.md) · [c3b experience](./milestone-c3b-director-experience-report.md) · [c3b quality](./milestone-c3b-director-quality-iteration.md) · [c3b model-ceiling](./milestone-c3b-model-ceiling-comparison.md) · [c3 close](./milestone-c3-close-report.md) | Story Director (design → eval → quality → close) | ✅ closed |
| [c3c ux-eval](./milestone-c3c-direct-ux-enhancements-eval.md) · [c3c completion](./milestone-c3c-direct-ux-enhancements-completion.md) | DIRECT UX (Skip Scan + alternatives) | 🟢 implemented |
| [c-eval-harness-plan](./milestone-c-eval-harness-plan.md) · [c3 dataset-plan](./milestone-c3-director-eval-dataset-plan.md) · [c3 direct-flow-audit](./milestone-c3-direct-flow-audit.md) | Eval harness + Director dataset + flow audit | planning/built |
| [c4-shot-validation-design](./milestone-c4-shot-validation-design.md) · [c4 go/no-go](./milestone-c4-go-no-go-audit.md) · [c4a eval](./milestone-c4a-shot-validation-eval-report.md) · [c4a.2 capture-evidence](./milestone-c4a2-capture-evidence-report.md) | Shot Validation design + eval (c4a: Bedrock AccessDenied for account) | 🟡 gate |
| [c4-lite-design](./milestone-c4-lite-capture-integrity-design.md) · [c4l-a heuristic](./milestone-c4l-a-heuristic-adoption-report.md) · [c4-lite completion](./milestone-c4-lite-completion-report.md) | C4-Lite capture integrity (A/B/C) | 🟢 built (C4L-D open) |
| [c3.5 figma-audit](./milestone-c3-5-figma-reconciliation-audit.md) · [c3.5 completion](./milestone-c3-5-completion-report.md) · [c35 completion](./c35-completion-report.md) · [c35 phase7 fidelity](./c35-phase7-fidelity-audit.md) | Figma UX convergence | ✅ device-accepted |

## Milestone D — Asset Understanding (real AU)

| Doc | What | Status |
|---|---|---|
| [d-au-experiment](./milestone-d-asset-understanding-experiment.md) | Intelligence/transport experiment | 🟡 experiment done |
| [d-au-ground-truth-owner-review](./milestone-d-au-ground-truth-owner-review.md) | AU ground-truth owner review | 🟡 AI-proposed |
| [d-au-implementation](./milestone-d-asset-understanding-implementation.md) | AU implementation + deploy | 🟢 deployed/verified |
| [d-photos-persistence](./milestone-d-photos-persistence.md) | Photos persistence (product-safety) | 🟢 implemented |

## Milestone E — Persistent Story Intelligence (Coverage · Guide)

| Doc | What | Status |
|---|---|---|
| [e-coverage-experiment](./milestone-e-story-coverage-experiment.md) · [e-coverage-reconciliation](./milestone-e-story-coverage-reconciliation.md) | Story Coverage experiment + design | 🔵/🟡 |
| [e2-guide-reconciliation](./milestone-e2-story-guide-reconciliation.md) · [e2a-guide-experiment](./milestone-e2a-story-guide-experiment.md) | Story Guide reconciliation + experiment | ✅ e2-a frozen |
| [e2b-coverage-migration-experiment](./milestone-e2b-story-coverage-migration-experiment.md) | StoryCoverage/2 migration experiment | 🟡 split verdict |
| [e2c-phase1-reconciliation](./milestone-e2c-phase1-reconciliation.md) · [e2c-closure](./milestone-e2c-closure-report.md) | E2-c closure | ✅ device-verified |

## Milestone Music (M-* ) — MusicPlan · MusicDirection · MusicAnalysis · Edit Planner · Epidemic

| Doc | What | Status |
|---|---|---|
| [music-reconciliation](./milestone-music-reconciliation.md) | Music design reconciliation | 🔵 design |
| [mb-musicplan-experiment](./milestone-mb-musicplan-experiment.md) | MusicPlan intelligence experiment | 🟡 experiment done |
| [mc-music-direction](./milestone-mc-music-direction.md) · [mc-closure](./milestone-mc-closure.md) | MusicDirection + Figma MUSIC | ✅ provisionally accepted |
| [md-music-sourcing-research-brief](./milestone-md-music-sourcing-research-brief.md) · [md2-api-verification](./milestone-md2-api-verification-edit-planner-reconciliation.md) | Music sourcing research + API verification | 🔵 research |
| [md3-editplan2-contract](./milestone-md3-editplan2-contract.md) · [md3-experiment](./milestone-md3-api-validation-edit-planner-experiment.md) | EditPlan/2 proposal + M-d.3 experiment | 🟡 experiment |
| [md-editplan2-freeze-epidemic-validation](./milestone-md-editplan2-freeze-epidemic-validation.md) · [md-epidemic-live-results](./milestone-md-epidemic-live-validation-results.md) | **EditPlan/2 FROZEN** + Epidemic live validation | ✅ frozen / validated |
| [md-musicanalysis-editplanner-adapter-impl](./milestone-md-musicanalysis-editplanner-adapter-impl.md) | MusicAnalysis freeze + production Edit Planner + adapter | 🟢 deploy-ready |

## Milestone R — Rendering (Media Engine → real Stories on device)

| Doc | What | Status |
|---|---|---|
| [ra-media-engine-feasibility](./milestone-ra-media-engine-v1-feasibility.md) | Media Engine V1 design + feasibility spike | 🔵 design |
| [rb-media-engine-impl](./milestone-rb-media-engine-v1-impl.md) · [rb-caseg-real-editplan](./milestone-rb-caseg-real-editplan.md) · [rb-closure-rc-design](./milestone-rb-closure-rc-design.md) | Media Engine V1 + Case G first real render | ✅ Case G device-accepted |
| **R-c** (in briefing + [09-edit-planner](./09-edit-planner.md)) | Consumer Create→Result + duration-feasibility preflight + executability repair (edit_plan/3) | ✅ device-accepted |
| **R-d** (in briefing; music docs above) | Music Integration: picker, music-aware render, soundtrack | ✅ music device-confirmed |
| [**R-d.1 editorial-timing + duration-intelligence**](./milestone-rd1-editorial-timing-and-duration-intelligence.md) | **edit_plan/4 significance-weighting (FROZEN+DEPLOYED)**; Duration Intelligence design+experiment | 🟡 at device gate / design |

## Experiments quick-reference (model-cost evaluations)

| Experiment | Doc | Result JSON (evals/results/) |
|---|---|---|
| Creative Intent eval | c1-creative-intent-eval-report | — |
| Scene Understanding | c2-scene-understanding-report | — |
| Director offline / quality / model-ceiling | c3a, c3b-quality, c3b-model-ceiling | — |
| DIRECT UX enhancements | c3c-direct-ux-enhancements-eval | — |
| Shot Validation corpus/model | c4a-shot-validation-eval-report | — |
| Capture Evidence transport | c4a2-capture-evidence-report | — |
| Asset Understanding intel/transport | d-asset-understanding-experiment | au_* |
| Story Coverage fixture/model | e-story-coverage-experiment, e2b-migration | story_coverage_* |
| StoryGuide model | e2a-story-guide-experiment | — |
| MusicPlan intelligence | mb-musicplan-experiment | — |
| Epidemic live API | md-epidemic-live-validation-results | — |
| **Epidemic MCP provider + energy_curve + adaptation** | [md5-epidemic-mcp-provider](./milestone-md5-epidemic-mcp-provider.md) | adapt review set (scratch, not committed) |
| **Track adaptation final quality gate (durations + A/B/C)** | [md5b-track-adaptation-quality-gate](./milestone-md5b-track-adaptation-quality-gate.md) | adapt review set (scratch) |
| **Music search + pagination (picker) · render-test removal** | [rd6-music-search-pagination](./milestone-rd6-music-search-pagination.md) | — |
| **Creative Editing V2-a Phase 0 (design/feasibility)** | [v2a-creative-editing-phase0](./milestone-v2a-creative-editing-phase0.md) | — |
| **Creative Editing V2-b P0 (boundary refinement, opt-in)** | [v2b-creative-editing-p0](./milestone-v2b-creative-editing-p0.md) | — (device A/B pending) |
| **Edit Planner max_tokens truncation fix (24-Moment 503)** | [edit-planner-maxtokens-truncation-fix](./milestone-edit-planner-maxtokens-truncation-fix.md) | — |
| **Native-audio disappearance audit (guitar performance)** | [native-audio-disappearance-audit](./milestone-native-audio-disappearance-audit.md) | — (fix proposed, owner gate) |
| **Next-page recommendations (music picker)** | [rd7-music-next-page-recommendations](./milestone-rd7-music-next-page-recommendations.md) | — |
| **Native-audio restoration (TMB+guitar) + V2-b immersive scope** | [native-audio-restoration-tmb](./milestone-native-audio-restoration-tmb.md) | — (device listening + V2-b gate) |
| **Edit Planner v5 native-audio preservation** | [edit-plan-v5-native-audio-preservation](./milestone-edit-plan-v5-native-audio-preservation.md) | FROZEN (dev default) — device-accepted |
| **Customize Your Story + sound preference** | [customize-your-story-sound-preference](./milestone-customize-your-story-sound-preference.md) | CLOSED — device-accepted |
| **M-e: My Music + Change Music + shared versioning** | [me-music-library-change-music-versioning](./milestone-me-music-library-change-music-versioning.md) | design done; implementing M-e (STOP at device gate) |
| Edit Planner M-d.3 (MusicAnalysis A/B) | md3-api-validation-edit-planner-experiment | edit_planner_md3.json |
| **Editorial Timing A/B (edit_plan/3 vs 4)** | rd1-editorial-timing… | edit_planner_editorial_timing_ab{,_v2}.json |
| **Guitar AU audit** | rd1-editorial-timing… | au_guitar_practice_001_audit.json |
| **Duration Intelligence (det vs semantic)** | rd1-editorial-timing… | duration_intelligence_{evidence,semantic}.json |

## Current position

- **Live on dev:** real Creative Intent, Scene, Director, AU, Story Coverage/2, Story Guide, MusicPlan,
  MusicDirection, MusicAnalysis, Edit Planner **edit_plan/4** (frozen), Media Engine V1.
- **At physical-iPhone gate:** edit_plan/4 editorial-timing acceptance (R-d.1).
- **Awaiting owner go:** Duration Intelligence implementation (fixes the 15s-default short-finale).
- **Internal-testing provider live:** Epidemic MCP (business key) selectable via `STORY_MUSIC_PROVIDER=mcp` —
  clears the 402 preview wall + populates energy_curve; track-adaptation experiment done, awaiting listening
  verdict (M-d.5). REST stays the production provider.
- **Deferred, not started:** Creative Editing V2 (natural-language re-cut).
- Nothing auto-pushed; local commits only.
| **P0: Concurrent Story processing (event-loop offload)** | [p0-concurrent-story-processing](./milestone-p0-concurrent-story-processing.md) | fix deployed to dev; device gate |
| **Editorial coverage: Guide wiring + mode-aware** | [editorial-coverage-guide-wiring-mode](./milestone-editorial-coverage-guide-wiring-mode.md) | wiring fixed; A/B done; mode gate for owner |
| **Edit Planner v5.1 editorial-mode (experiment)** | [edit-plan-v5-1-editorial-mode](./milestone-edit-plan-v5-1-editorial-mode.md) | deployed to dev; freeze gate for owner |
| **AU evidence & visual distinctness (girl_with_dress)** | [au-evidence-visual-distinctness](./milestone-au-evidence-visual-distinctness.md) | A/B/C done; forwarding fix for owner |
| **Creative editorial freedom & user-directed editing (study)** | [creative-editorial-freedom-architecture](./milestone-creative-editorial-freedom-architecture.md) | A/B done; architecture gate for owner |
| **V2-b.1 — Edit Planner v5.2 candidate & device gate** | [v2b1-edit-planner-v5.2-candidate-and-device-gate](./milestone-v2b1-edit-planner-v5.2-candidate-and-device-gate.md) | Candidate promoted (commit 1598949); freeze pending on-device render; default stays v5.1 |
| **Music-fill correctness + music-aware pacing (design)** | [music-fill-and-pacing-alignment](./milestone-music-fill-and-pacing-alignment.md) | Music-fill implemented+tested (commit 8b65b7f), device gate pending; pacing = design + bounded experiment, STOP at architecture gate |
| **Multimodal creative editing: visual evidence + director reasoning** | [multimodal-creative-editing-visual-evidence-experiment](./milestone-multimodal-creative-editing-visual-evidence-experiment.md) | **A1 richer AU forwarding IMPLEMENTED** (app 78da143, iOS-only, device gate pending); B stays experimental (mixed: won hiking, regressed guitar phrase); Arm C NOT justified (errors are reasoning/planning, not frame density); next lever = multi-segment-per-Moment planning. **Creative-Objective Exp 1 (short-form social, 2026-10-13):** largest quality lever = explicit creative brief + bounded critique/revision (Arm D), NOT multimodal frames — D fixes the hook & finds omitted/repetitive footage; brief alone kills clip-reuse; images did not improve the 3s hook and cost most ($7.44/$20, 17/20 cells, 3 quota-blocked). No prompt/contract change; nothing pushed |
| **MVP Multimodal Creative Editor Integration (+ adaptive visual budget)** | [mvp-multimodal-creative-editor-integration](./milestone-mvp-multimodal-creative-editor-integration.md) | **IMPLEMENTED, device-acceptance pending.** New `POST /v1/edit-plan-mm` multipart endpoint attaches bounded Tier-1 contact sheets + versioned creative brief (`shortform_mvp/1`) to the SAME planning call via shared core with frozen `/v1/edit-plan` (same validator/semantics); gateway `images` surface, frozen v5.2 body untouched, images ephemeral. iOS: contact-sheet builder + adaptive visual-budget preflight (resize→compress→fewer-frames→reduce-coverage, no model call) + `planMultimodal` with graceful text-only A1 fallback, wired behind `StoryMultimodalPlanning` flag (default OFF). Backend 40/40 edit-planner tests (6 new MM); iOS parse-clean (Xcode type-check + Phases 4-5 device acceptance owner-gated). No contract change; nothing pushed/deployed (app 924d22f + iOS) |
| **MVP Customize UX: Editing Approach selection** | [mvp-customize-ux-editing-approach](./milestone-mvp-customize-ux-editing-approach.md) | **IMPLEMENTED (iOS-only), device-acceptance pending.** New *Editing approach* section on Customize screen (Storytelling / Visual-first), gold-card UX per Figma, no jargon. Persisted `StoryModel.editingApproachExplicit` (additive/migration-safe, never overwritten on retry). Routing: Storytelling→`/v1/edit-plan`, Visual-first→`/v1/edit-plan-mm`, BOTH gated by `StoryMultimodalPlanning.enabled` (option hidden when flag off; coordinator defends too). Fallback transparency: diagnostics record requested-vs-actual approach, never claims visual analysis that didn't happen. Beacon `editplan: req=/actual=`. `EditingApproachTests` added; frozen prompts/EditPlan2/Music/Media untouched. Nothing pushed |
| **Async Edit Planning + Production Rollout plan** | [async-edit-planning-and-prod-rollout](./milestone-async-edit-planning-and-prod-rollout.md) | **Async backend+iOS DONE & dev-accepted** (removes App Runner's ~120s ceiling: instant job id → Lambda worker runs frozen edit_plan/5.2 plan→critique→validate up to 15 min; DynamoDB/SQS/S3/Lambda/EventBridge via env-parameterized CDK; iOS `EditPlanJobClient` sole creation path + honest Creating UX). **iOS "defaults" audit:** async + multimodal + visual-first already ON by default; critique + post-plan refinement intentionally OFF (owner rule) — flag-flip request needed NO code change. **Prod rollout ON HOLD** — blockers: owner's prod account id/profile, explicit deploy authorization, git-push decision. Full preconditions/steps/rollback/risks documented. Nothing deployed to prod; nothing pushed. Known gap: fallback gateway swallows the ProviderError reason (recommend logging `last_error`) |
| **MVP Reconciliation — Create Now · Versions · NL Revision** | [mvp-reconciliation-createnow-versions-nlrevision](./milestone-mvp-reconciliation-createnow-versions-nlrevision.md) | **AUDIT + ARCHITECTURE PROPOSAL, awaiting owner approval.** Code-grounded (3 parallel read-only audits, file:line). Create Now: NO wired entry (Home card `enabled:false`) — pure composition over existing engines, ≤2-decision flow + ephemeral Story. Create Another Version: root cause of identical versions = temp 0.0 + no prior-plan input + idempotency cache; fix = one additive `prior_plans`/`avoid` field into the USER message (evidence-driven diversity, no shuffle/templates); lineage data model already exists. NL Revision: NONE exists today; reuse the critique→revise→validate backbone; soft prefs = wiring, hard directives (include/exclude/opening/ending/order) = new `directives` object + new `validate_plan` predicates (never silent). Order: C → A → B. Frozen EditPlan/2 + edit_plan/5.2 untouched (all additive). STOP at approval; nothing pushed/deployed |
| **Capture Anywhere + Global Moments Library** | [capture-anywhere-global-moments-library](./milestone-capture-anywhere-global-moments-library.md) | **ARCHITECTURE GATE, awaiting owner approval. STOP before changing persistent models.** Code-grounded audit: today Story owns Moments via `.cascade` and `MomentModel.story` is SINGLE-valued → deleting a Story deletes its Moments+media (the behavior to remove). Proposal: independent Moments, many-to-many `MomentModel.stories` ⇄ `StoryModel.moments`, Story→Moment delete rule `.nullify` (media follows the Moment, never deleted with a Story). `MomentUnderstandingModel` is ALREADY global (keyed by momentId) → AU needs no re-scoping; Story-specific interpretation derived at plan time, not stored. Migration: additive-bridge (add many-to-many, copy `story`→`stories`, keep old to-one one release) preferred over fragile custom SwiftData migration; device-gated. Capture reuses existing Shoot-a-Moment (no new engine); Create-from-Moments reuses the existing pipeline (no media dup, no 2nd Edit Planner). 8 acceptance tests. Supersedes the ephemeral-Story Create Now (C). Nothing pushed/deployed |
