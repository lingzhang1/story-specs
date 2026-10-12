# Milestone M-e — My Music + Change Music, on a Shared Versioning Foundation

Status: **DESIGN + IMPLEMENTATION (M-e only).** Create Another Version (V2-b) and
Natural-Language Refinement (V2-c) are DESIGN-ONLY here (Phase 6). Owner gate:
STOP at the M-e physical-device acceptance. Do not push automatically. Do not
change frozen contracts without explicit owner approval.

Preceding accepted milestone: Customize Your Story + sound preference
(`milestone-customize-your-story-sound-preference.md`), device-accepted on build
`b6c0237-guidewire+ensure` (keep15/duck6/mute0 on TMB, owner listened + confirmed
the native/music balance). That milestone is **CLOSED** by this document (Phase 0).

---

## Phase 0 — Customize Your Story closure

Verified at HEAD `b4befdf`:
- `af5ae11` froze `edit_plan/5` as the development default (v4 preserved + reachable via `STORY_EDIT_PLAN_VERSION=4`); EditPlan/2 output schema and renderer gains untouched.
- `9a9d8a8` wired `sound_preference` (immersive|balanced|music_led) through `/v1/music-plan` + `/v1/edit-plan` (additive request field, default None) and shipped `CustomizeYourStoryView`.
- `6e61587` compacted the Full Story row; `b4befdf` removed the DEBUG audio diagnostics readout.
- Backend suite: 224 passing. iOS: parse-clean (device build is the real gate, and it passed).
- **Owner physical-device acceptance recorded.** The native-audio regression is accepted as resolved; renderer gain table and audio prompts are frozen for this milestone (owner: do not tune further).

Working tree: clean except untracked `evals/scenarios/guitar_practice_001/*.mov` fixtures (unrelated media — preserved, never committed).

---

## Phase 1 — Persistence / contract reconciliation (THE load-bearing finding)

### What is persisted today for a generated video (`GeneratedOutputModel`)
`id`, `story`, `title`, `durationSeconds`, `momentIds` (legacy) + `sourceMomentIds`
(ordered), `sourceFingerprint`, `editPlanProvenance` (a SHORT string:
`schema_version; model=none|music; planned=Ns`), `relativeFilePath` (the rendered
MP4 under `<Documents>/rendered/`), `outputFormat`, `renderWidth/Height`,
`isStale`, `soundtrackTitle`, `createdAt`.

### Can the exact visual edit be reconstructed today? **NO.**
The executable plan (`EditPlanV2`: per-segment `order`, `assetId`, source in/out,
`outputDurationSeconds`, `speed`, `transition`, `nativeAudio`, `musicAlignment`)
is **not persisted anywhere**. The raw backend plan JSON is decoded at
`EditPlannerService.swift:107` and discarded; `EditPlanV2` is `Decodable`-only
(cannot even round-trip). The plan is also potentially mutated post-decode by the
opt-in `CreativeRefinementV2b` source-nudge pass, so even re-calling the Edit
Planner would NOT reproduce the same timeline. `editPlanProvenance` is a human
breadcrumb, not an executable artifact.

**Conclusion:** Operation A (Change Music — "same visual edit, same moment
selection/order, same source in/out, same speed/crop/transitions, same final
duration, different soundtrack") is **impossible on existing persistence.** This
is the architecture gap Phase 1 asked me to find.

### Is it a "significant" gap requiring a full STOP? No — it is bounded and additive.
The owner's Phase 1 explicitly asks: "Investigate whether existing
GeneratedOutput/SwiftData structures can be **extended rather than replaced**." The
answer is **yes, extend.** No frozen contract changes: EditPlan/2's *wire schema*
is untouched; we persist a local *instance* of a plan the backend already emits.
So I proceed to implement M-e on the extension below, rather than stopping — the
gap's fix IS the shared-versioning deliverable.

### Shared versioning foundation — EXTEND `GeneratedOutputModel` (do not replace)
A `GeneratedOutputModel` row already IS a version (a Story has many, additively,
never overwritten). I add version lineage + the executable plan to it. All new
fields optional/defaulted → SwiftData lightweight migration, existing rows stay
valid (legacy rows read as v1-of-their-story, operation `original`, with no stored
plan — correctly NOT eligible for Change Music, which needs the plan).

New fields on `GeneratedOutputModel`:
- `versionId: UUID` (default = `id`; the stable version handle)
- `parentVersionId: UUID?` (nil for an original render; set to the source output's `versionId` for a derived version)
- `operationType: String` = `original` | `change_music` (future: `another_version`, `edit_ai`)
- `editPlanJSON: String?` — the **exact executable `EditPlanV2`** this output rendered from, as canonical snake_case JSON (the post-refinement plan). nil on legacy rows. This is what makes the visual edit reconstructable.
- `soundtrackRole: String?`, `nativeAudioPriority: String?` — the mix knobs used, so a re-render reproduces the same native-audio treatment.
- `soundPreferenceRaw: String?` — the effective sound preference at render time (provenance).
- `musicAssetRef: String?` — identity of the soundtrack: `provider:<trackId>` | `mymusic:<MyMusicAsset.id>` | `none`. (`soundtrackTitle` is kept for display.)

`EditPlanV2`/`EditSegmentV2`/`Transition`/`MusicAlignment` become **`Codable`**
(add `Encodable`) so the post-refinement plan can be serialized at persist time and
re-decoded for a re-render. Round-trip stability is unit-tested.

Reconstruction for Change Music (Operation A): decode `editPlanJSON` → `EditPlanV2`,
resolve each `assetId` → on-device URL via `MomentResolver` (Moments/assets still
live under the Story; **no duplicate source video is stored** — versions share the
originals), swap only the soundtrack + mix knobs, re-render through the EXISTING
`MediaEngineV1`. The Edit Planner is **never invoked**. Visual timing is byte-for-byte
the same plan.

### Does `MusicAsset` exist? No — introduce `MyMusicAssetModel` (Phase 2)
Provider tracks are resolved transiently per-create (`MusicSelection`), never
persisted as a library. My Music needs a new persisted `@Model`.

---

## Phase 2 — My Music library (`MyMusicAssetModel` + `MyMusicStore` + import)

`@Model MyMusicAssetModel`: `id: UUID` (stable identity), `displayName`,
`relativePath` (under `<Documents>/mymusic/`, LOCAL-first — bytes never leave the
device, never uploaded), `originalFilename`, `durationSeconds`,
`contentHash` (sha256 of bytes — dedup + corruption detection), `energyCurveJSON`
(coarse amplitude, locally derived), `bpm: Double?` (ONLY if validated locally;
never fabricated), `createdAt`, `lastUsedAt?`.

Import: SwiftUI `.fileImporter` (iOS Files), unprotected audio UTTypes
(mp3/m4a/wav/aiff/caf). Copy bytes into managed storage, hash for dedup (same hash
→ reuse existing asset, no second copy), reject DRM/unreadable with a clean error.
Preview (AVPlayer, `.playback` category per the known AVAudioSession lesson),
rename, reuse across Stories, remove (deletes managed bytes + row; a Story version
referencing a removed asset degrades gracefully to no-music on re-open, never
crashes).

Local analysis (`LocalMusicAnalysis`): duration via `AVURLAsset.load(.duration)`;
coarse energy curve by RMS-windowing decoded samples (`AVAudioFile`), normalized
0..1 (reuses the frozen `normalize_energy` contract shape); beats/bpm ONLY if a
validated local estimate is available, else honestly empty. **No fabricated
BPM/beats/downbeats/highlights.** Produces a `MusicAnalysisV1`-shaped object so the
existing mix path consumes it unchanged.

My Music is clearly labelled distinct from provider tracks in the picker. Provider
favorites are NOT turned into permanent local copies (licensing) — only
user-imported bytes are stored.

---

## Phase 3 — Change Music (new version, no Edit Planner)

On the Result screen, a "Change Music" action opens a chooser (reuse the Music
Picker visual language + a My Music section): pick another provider track, a saved
My Music track, a newly imported track, or **no music**. On confirm:
1. Load the source output's `editPlanJSON` → `EditPlanV2` (the exact visual edit).
2. Resolve the new soundtrack: provider → temporary authorized download; My Music → managed local file; none → nil.
3. Fit music to the EXISTING video duration with a bounded trim + fade policy (no visual cut changes to chase the beat — that is a future op).
4. Re-render via `MediaEngineV1` with the SAME plan, preserving native audio + the stored sound preference / role / priority.
5. Persist a NEW `GeneratedOutputModel` version: `parentVersionId` = source, `operationType=change_music`, copy `editPlanJSON`, set the new `musicAssetRef`/`soundtrackTitle`. **Previous version is preserved** (never overwritten).

---

## Phases 4–5 — Regression + device acceptance
Regression (Phase 4) and the physical-device checklist (Phase 5) per the owner's
enumerated lists. Soundtrack-only revisions are verified to preserve the EXACT
visual timeline (same plan JSON), not merely a similar edit. STOP at device gate.

## Phase 6 — V2-b / V2-c roadmap (design only; see end of this doc after M-e lands)
`another_version` and `edit_ai` reuse the SAME version lineage
(`operationType`, `parentVersionId`, `editPlanJSON`). V2-b reads prior
`editPlanJSON`(s) to steer AWAY from used moments/ranges/openings; V2-c resolves
NL references against ALL MomentUnderstanding (incl. omitted footage) and edits the
stored plan minimally. Neither is implemented in M-e.

---

## Contract implications
- No frozen wire-contract change. EditPlan/2, MusicPlan/1, MusicAnalysis/1 schemas untouched.
- `EditPlanV2` renderer DTO gains `Encodable` (local round-trip only).
- `GeneratedOutputModel` + new `MyMusicAssetModel`: additive SwiftData migration, all new fields optional/defaulted.
- Change Music is a CLIENT-side re-render — no new backend endpoint.

## Risks / limitations
- My Music local BPM/beat estimation is best-effort; absence is honest-empty, mix degrades to energy-only (same graceful path as provider tracks lacking beats).
- A My Music asset removed after a version referenced it → that version re-opens as no-music (bytes gone); documented, not a crash.
- Licensing: provider tracks are never permanently stored; only user-owned imports are.
