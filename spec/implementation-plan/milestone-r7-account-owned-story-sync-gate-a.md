# R7 — Account-Owned Story Metadata Synchronization (Gate A)

Status: 🟡 GATE A — architecture + code review. NOTHING provisioned, NOTHING deployed, no code merged on the
basis of this doc. STOP before provisioning; owner approval opens Gate B. Builds on the R7 Phase 0 audit.

Owner decisions locked: (1) optimistic concurrency with a server monotonic revision — NO silent LWW;
(2) back up completed rendered MP4s as account-owned cloud assets (reuse the R2–R6 media infra); (3) sync on
change + foreground + post-auth + full-pull on a new device, debounced, offline-retried, idempotent,
account-scoped. App repo iOS work on `feat/r1-account-ownership-isolation`; backend ships via the approved
master-only PR → `deploy-dev.yml` pipeline.

---

## 0. Audit answer — can the existing media service back up generated MP4s safely?

**Yes, with the smallest additive change being an optional `media_kind` tag.** `cloud_media_id_for(owner,
content_sha256) = uuid5(owner + sha256)` and the key `media/{owner}/{cloud_media_id}/original` are
**content-addressed + owner-scoped** — nothing is Moment-specific. A generated MP4 is bytes with a sha256
owned by an account, so it reuses the EXACT initiate/presign/verify/download path with **no weakening** of the
Moment ownership or cloud-state invariants:
- Ownership still derived from the trusted `principal.story_user_id`; never client-supplied.
- Account-scoped key + per-account `uuid5` salt → no cross-account dedup or leak (decision-2 #9).
- Verify = exists ∧ size ∧ full-file sha256 (single) / composite (multipart) — unchanged (decision-2 #8).
- Download = presigned GET, verified-only, account-scoped — unchanged (decision-2 #7).

**Smallest additive change (proposed for approval):** add an OPTIONAL `media_kind` attribute to the media
record — `"original"` (default when absent, so every existing record is unchanged) or `"generated"`. It is a
TAG for listing/provenance only; it gates no logic and changes no invariant. No versioned-contract break.
(A generated output keeps its own `source_story_id` + `source_output_id` in the OPTIONAL tag fields that
already exist alongside `source_moment_id`.)

---

## 1. Cloud Story document schema (DynamoDB)

A new table `story-<env>-stories`, SEPARATE from the media table (different lifecycle, different access
pattern). One item = one Story document for one account.

```
PK  story_user_id   (S)   -- canonical account; every read/write scoped to principal.story_user_id
SK  story_id        (S)   -- the Story's UUID (SAME id cross-device; no remap)
    revision        (N)   -- SERVER-assigned monotonic revision. Starts 1; +1 on every accepted write.
    deleted         (BOOL)-- soft-delete tombstone (so a delete syncs; bytes/cloud media untouched)
    updated_at      (N)
    doc             (S)   -- JSON: the Story document body (below), schema_version "story_doc/1"
    incomplete_deps (BOOL)-- true if any referenced Moment is not yet cloud_verified (decision: honest backup)
```

`doc` (JSON, `story_doc/1`) — ESSENTIAL creative state only (no raw local paths — owner rule):
```jsonc
{
  "schema_version": "story_doc/1",
  "story": { "id, title, status, intent{theme,tone,summary,subject,desired_feeling[],style_tags[],
             emphasis[],constraints[],pace,target_duration_seconds}, sound_preference_explicit,
             editorial_mode, editing_approach_explicit, creative_directives (resolved {must_include/
             exclude/opening/ending/order_*} of STABLE Moment UUIDs), creative_direction_text,
             plan_operation_id, plan_job_id, created_at, updated_at" },
  "moment_refs": [ { "moment_id", "order_index", "added_at",
                     "cloud_media_id|null", "cloud_verified": bool,
                     "caption", "source",  // Moment RECORD essentials — NOT the bytes, NOT a local path
                     "captured_at|null", "latitude|null", "longitude|null" } ],
  "music_direction": { "direction_json", "user_authored" } | null,
  "story_guide": { ...required creative state... } | null,
  "outputs": [ { "output_id", "title", "duration_seconds", "source_moment_ids[]",
                 "source_fingerprint", "edit_plan_provenance",
                 "cloud_media_id|null",      // the rendered MP4's media record (decision 2)
                 "cloud_verified": bool } ]
}
```
Rules: raw `relativePath` is NEVER written (owner rule — not an authoritative cloud reference). A Moment ref
carries `cloud_media_id` only when that Moment is cloud-backed; otherwise `cloud_verified:false` and the doc's
`incomplete_deps=true` (honest: "backed up except N Moments", never "fully backed up").

---

## 2. Backend APIs (all `require_user`, account-scoped to `principal.story_user_id`)

- `GET  /v1/stories` — the account's Story INDEX: `[{story_id, revision, updated_at, deleted, incomplete_deps}]`.
  Full-account pull on a new device (decision 3). No bodies (cheap); client pulls bodies it lacks/stale.
- `GET  /v1/stories/{story_id}` — the full document + its current `revision`.
- `PUT  /v1/stories/{story_id}` — upsert with **optimistic concurrency**. Body: `{base_revision, doc,
  deleted?}`. CONDITIONAL write: `attribute_not_exists(story_id)` (create, base_revision 0) OR
  `revision = :base_revision` (update). On success → new `revision = base+1`, returns it. On mismatch → **409
  with the CURRENT {revision, doc}** so the client can run Keep-Both (decision 1 #3/#4/#5).
- `POST /v1/stories/{story_id}/outputs/{output_id}/backup` — register a completed render's `cloud_media_id`
  on the output (after the MP4 is uploaded + verified via the EXISTING media endpoints). Refuses an
  unverified cloud_media_id (decision 2 #5/#8).

Generated MP4 bytes use the ALREADY-DEPLOYED media endpoints unchanged:
`/v1/media/initiate|multipart/*|{id}/complete|{id}/download` with `media_kind:"generated"`. No new byte path.

---

## 3. Optimistic concurrency (decision 1 — the smallest understandable model)

- Server owns `revision` (monotonic per Story). The client stores the last `revision` it observed per Story.
- Every `PUT` carries `base_revision`. DynamoDB `ConditionExpression`:
  `attribute_not_exists(story_id) OR revision = :base` → a stale client is REJECTED, never overwrites (#3/#5).
- **409 conflict response** returns the server's current `{revision, doc}`. The client:
  1. KEEPS its local conflicting draft (never discards — #6),
  2. offers **Keep Both / Save as Another Version**: clone the local draft as a NEW Story (`new story_id`,
     title "… (copy)") and `PUT` it at base_revision 0, then adopt the server's version for the original
     `story_id` (#7).
- NO automatic field merge, NO version vectors (#8). Delete = soft tombstone synced the same way.

---

## 4. Generated MP4 backup integration (decision 2)

On a render completing (`GeneratedOutputModel` with a `relativeFilePath`, state complete):
1. Enroll the MP4 via a GENERATED-kind upload (reuse `MediaUploadQueue`/`MediaUploadService` with
   `media_kind:"generated"`, `source_story_id`, `source_output_id`) — streamed sha256, single or multipart by
   size, server-verified. Failed/incomplete renders are NOT enrolled (#5).
2. On verify, stamp `GeneratedOutputModel.cloudMediaId` + a `cloudVerified` flag (NEW additive fields,
   mirroring the Moment pattern), and the next Story sync carries it.
3. Device B: a synced output with a `cloud_media_id` restores the MP4 for playback/sharing via the EXISTING
   R6 `MediaResolver` path (download → verify full-file sha → install) — **no AI re-planning, no re-render**
   (#6). Lineage (`source_moment_ids`, `source_fingerprint`, `edit_plan_provenance`), music, and edit-plan
   provenance travel in the doc (#4).

Additive iOS model fields: `GeneratedOutputModel.cloudMediaId: String?`, `cloudVerifiedAt: Date?` (defaulted
nil — lightweight migration). No frozen-contract touch.

---

## 5. iOS sync coordinator design

`StorySyncCoordinator` (`@MainActor`, actor-coalesced like `MediaResolver`):
- **Local cache stays authoritative for editing** — all edits write SwiftData first; sync is async + never
  blocks the editor (decision 3: no network dependency for local editing).
- **Push**: on a Story change, debounce (~2–3 s) then `PUT` the doc at its stored base_revision. A persistent
  `StorySyncJobModel` (per `(owner, story_id)`, like the upload/download job rows) survives relaunch and
  retries offline (decision 3). Idempotent: re-pushing an unchanged doc at the current revision is a no-op.
- **Pull**: on foreground / post-auth / new-device → `GET /v1/stories` index, then `GET` each story whose
  server `revision` > local; apply into SwiftData under the active account only (account-scoped; never writes
  another account's rows — #7 isolation). A pulled Moment ref with a `cloud_media_id` but no local Moment
  creates a **cloud-only Moment shell** (record + `cloudMediaId`, `localAvailability=.evicted`) that R6
  restores on demand — this is also the fresh-install gap the R6 reconcile couldn't close.
- **Conflict**: a 409 routes to the Keep-Both path (§3) and surfaces a non-blocking notice; never silent.
- **Honesty**: a Story with `incomplete_deps` shows "Backed up (N moments pending)" — never "fully backed up"
  (owner rule / R6 #12).

---

## 6. Migration strategy (lossless — decision requirement)

- Existing Stories already have UUID ids + `ownerAccountId`; no id remap. First sync after R7 ships: the
  client pushes each owned Story at `base_revision 0` (create) → server assigns revision 1. Idempotent, so a
  re-run is safe.
- Additive SwiftData fields only (`GeneratedOutputModel.cloudMediaId/cloudVerifiedAt`, `StorySyncJobModel`,
  `media_kind` is backend-only) → SwiftData lightweight migration; existing stores decode cleanly. (NOTE: a
  schema add triggers a migration — per the R6 lesson, verify existing rows keep their state and the migration
  does not reset defaults; the R6 server-reconcile already self-heals Moment cloud-state.)
- No existing Moment/output bytes move. Generated-MP4 backup is opt-in per completed render going forward;
  a one-time "back up existing versions" pass can enroll historical outputs (bounded, user-initiated).

---

## 7. Security tests (Gate A deliverable; run before Gate B provisioning)

- A `PUT`/`GET` for a Story NOT owned by the caller → 404 (never another account's doc). Cross-account index
  isolation (A's `/v1/stories` never lists B's).
- Conflict: stale `base_revision` → 409 with server doc; server state unchanged; no silent overwrite.
- Generated-output backup: cannot register an UNVERIFIED cloud_media_id; cannot register another account's
  cloud_media_id; `media_kind` tag never bypasses ownership.
- Idempotent `PUT` at current revision = no-op (no revision inflation).
- Soft-delete syncs without touching cloud media.
- iOS: account switch mid-sync never writes B's rows under A (account-scoped execution).

## 8. Engineering estimate
Backend (stories table + 4 endpoints + conditional writes + media_kind tag + tests): ~2–3 d. iOS
(StorySyncCoordinator + StorySyncJobModel + doc encode/decode + conflict Keep-Both UX + generated-output
enroll/restore + migration + tests): ~4–6 d. Combined two-device acceptance: ~1 d. Total ≈ 7–10 eng-days.

## Explicit exclusions
No R4-C background uploads, no permanent cloud media deletion, no billing, no production, no CI/CD redesign,
no frozen editing-contract change (the only contract touched is the NEW `story_doc/1` + the additive
`media_kind` tag, both proposed here for approval). STOP before provisioning (Gate B needs owner approval).
