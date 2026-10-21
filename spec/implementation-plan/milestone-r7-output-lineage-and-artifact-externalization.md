# YoTale R7 — Output Version-Lineage Decode Fix + ai_artifact Externalization

Status: **IMPLEMENTED (iOS + backend); backend DEPLOYED + live-verified (dev); AWAITING two-device physical acceptance.**
Branch: `feat/r1-account-ownership-isolation`. Commits: `81bac08` (lineage decode fix), `e90ea13` (Phase 4 externalization).
Worktree: `/Users/lingzhang/Workplace/story`.

## Why this milestone exists

A two-device test of the TMB Story showed Device B regenerating the Story Guide and starting a **fresh** Story
("Create my Story" with no versions) despite the R7 sync client being present. Reconciliation (below) proved the
code was on the right branch and compiled in — the defect was a **missing decode path**, not missing code.

## Phase 0 — Reconciliation (previous vs current R7 code)

A prior diagnosis ("no iOS Story API calls exist") was run in the WRONG worktree
(`/Users/lingzhang/Workplace/story-wt-r7d-generated-mp4-backup`, branch `feat/r7d-generated-mp4-backup` @
`a0ea45f` = the backend-only R7 merge). The full R7 iOS sync client exists on
`feat/r1-account-ownership-isolation` @ `9d750ed`: `StorySyncService` / `StorySyncCoordinator` (`233b21a`,
`38c072c`), `StoryDocCodec` + `story_doc/2` derived state (`62001b5`), `UnderstandingSync`, `CloudHydrationGate`
(`8cd4504`), R7-D MP4 backup (`9d750ed`). Target membership verified: the `Story` group is a
`PBXFileSystemSynchronizedRootGroup` whose only exclusion is `Info.plist`, so every `Media/Sync/*.swift`
compiles into the app. No uncommitted sync-source changes existed to clobber.

## The defect (root cause of the Device-B symptoms)

1. **`StorySyncCoordinator.apply()` never created `GeneratedOutputModel` rows.** It decoded `story` + `moment_refs`
   only; nothing materialized the doc's `outputs[]` array on a new device. `applyDerived`'s `output_edit_plans`
   loop only *updated an output that already existed locally*. On Device B the outputs never existed → zero
   versions → "Create my Story" started fresh.
2. **`StoryDocCodec.encode` dropped version lineage** from `outputs[]` (no `version_id`, `parent_version_id`,
   `operation_type`, render shape, `soundtrack_title`, music refs).
3. The "different Guide" was a downstream consequence: the `CloudHydrationGate` ordering is correct
   (`StoryHomeView.loadCoverage` waits for hydration before `ensureGuide`); the Guide is restored by
   `applyDerived(preferCloud:true)` once the pull runs on a current binary.

## Fix (commit `81bac08`)

- `StoryDocCodec.encode`: `outputs[]` now carry full lineage.
- `StoryDocCodec.applyOutputs` (new): idempotent create of `GeneratedOutputModel` rows from `doc.outputs`
  (keyed by stable `output_id` → no duplicates on re-pull), preserving lineage, restoring the exact
  `editPlanJSON` from `derived.output_edit_plans` and the MP4 `cloudMediaId`/`cloudVerifiedAt` for R6
  restore-on-play. `relative_file_path` from the cloud is a hint only — never adopted as authoritative; R6
  re-proves bytes before playback. NEVER re-renders.
- `StorySyncCoordinator.pull()` and `resolveKeepBoth()` call `applyOutputs` before `applyDerived`.
- `StoryCreativeRefinement.buildIdentifier` bumped `me-mymusic-top+save` → `r7-output-sync-1` for device
  build-id proof (shown in the Music picker).

## Phase 4 — ai_artifact externalization (commit `e90ea13`)

The backend caps a Story document at 350 KB and FAILS explicitly (413) rather than truncating. A Story with
many large validated EditPlans must externalize the oversized plans so version history is never capped.

- **Backend** (`packages/media_store`): add `MEDIA_KIND_AI_ARTIFACT`. No new bucket, table, or endpoint — an
  EditPlan is immutable, content-addressed bytes, so it rides the EXISTING R4 upload / R6 verified-download
  pipeline (same key `media/{owner}/{cmid}/original`, `cloud_media_id = uuid5(owner+sha256)`, full-file SHA256
  verify, non-deleting retention). `content_type` on the media initiate path is a free string, so an artifact
  declares `application/json`. +4 backend tests.
- **iOS `AIArtifactStore`** (new): `externalize(json)` uploads + verifies and returns a durable reference
  `{cloud_media_id, sha256, bytes, schema_version}`; `resolve(ref)` downloads with size+SHA256 verification and
  re-checks the declared sha, throwing explicitly if an artifact cannot be restored (never drops a version).
- **`StoryDocCodec`** pure transforms: `plansToExternalize` (over-budget → largest plans first),
  `applyExternalizedRefs` (swap inline JSON → `edit_plan_ref`), `externalizedRefs` + `inlineResolvedPlans`
  (resolve back before decode). Refs are added to `referenced_cloud_media_ids` for the server C3 check.
- **`StorySyncCoordinator`**: externalize before the PUT when over budget (a 413 remains the safety net);
  resolve refs before decode on pull and conflict-adopt; on failure the doc stays dirty / the conflict is kept
  for retry — a version is never decoded with a missing plan.

## Requirement coverage (owner Phases 1–5)

- Phase 1 (StoryDocCodec): full lineage encode/decode incl. version_id/parentVersionId/operationType, exact
  EditPlan, MP4 cloud ref + verification state; never a local path as a cloud reference.
- Phase 2 (StorySyncService): existing debounced push / offline retry / monotonic revision / payload-bound
  mutation_id / 409 Keep-Both-or-Use-Cloud / account isolation / idempotent re-pull (output rows now dedup by
  output_id) — unchanged except the decode completion.
- Phase 3 (AI-state hydration): existing `applyDerived` + `UnderstandingSync` + `CloudHydrationGate` restore
  Guide/Coverage/Music/duration/AU and EditPlans before any generation; this milestone makes the EDIT PLANS +
  VERSION ROWS part of that hydration real.
- Phase 4: externalization above (no silent cap/truncation; content-addressed; account-scoped; SHA256-verified;
  durable refs; no deletion; explicit failure).
- Phase 5 (Generated MP4): R7-D path (commit `9d750ed`) unchanged — upload via R4 queue, cloud_verified before
  the ref is committed, lineage preserved, cloudMediaId synced, R6 restore-on-play, never re-renders.

## Tests

- Backend: `apps/api/tests` + `packages` FULL suite **438 green** (media API 48, incl. +4 ai_artifact).
- iOS: `StorySyncTests` extended — encode lineage, create-on-pull, idempotent re-pull, exact-plan restore,
  full-pull materialization, owner isolation (lineage fix) + Phase-4: small-doc-externalizes-nothing,
  largest-first + inline round-trip, artifact-store roundtrip-verified, integrity-failure throws,
  push-externalizes-oversized, pull-resolves-externalized-into-version.
- Compile gate (honest): the Kiro sandbox blocks `swift-plugin-server`, so `xcodebuild`/XCTest cannot run here
  (every `@Model` fails "external macro could not be found" — an environment limit, not a code error). All
  touched files are `swiftc -parse` clean and the pure Phase-4 codec transforms compile+run via standalone
  `swiftc`. The authoritative Xcode compile + XCTest run on the device/developer machine.

## Deploy

CodeBuild `story-rb-caseg-build` SUCCEEDED → App Runner `story-development-api` forced `start-deployment` →
RUNNING, `/health` 200. Running container's last log event (00:16:18) is after the ECR `:latest` push
(00:15:06), confirming a FRESH digest (not stale). No new env, no DynamoDB table change (media table is
schemaless; the new `media_kind` value and artifact records need no migration).

## Remaining (the STOP point)

Two-device physical acceptance on the same account: Device A (Guide/Coverage/AU present, existing versions,
MP4s cloud_verified, sync bumps the backend revision) → Device B (sign in, pull, identical Story ID + Moment
order + Guide + Coverage, versions appear in the Result/version UI with exact EditPlans + lineage, a restored
MP4 plays through R6, **zero** restoration-triggered Bedrock calls, no duplicate rows after relaunch, another
account cannot see the Story). The device Xcode build is the authoritative Swift compile gate.

## Boundaries honored

Dev backend + existing CodeBuild/App Runner pipeline only. No prod deploy, no billing, no CI/CD redesign, no
frozen EditPlan/2 or prompt change, no add-new-Moment-during-revise (deferred P1), no expensive Bedrock jobs,
no destructive local migration, all existing media/versions preserved.
