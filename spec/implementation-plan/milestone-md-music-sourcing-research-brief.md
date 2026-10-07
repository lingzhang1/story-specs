# Milestone M-d — Music Sourcing & Analysis: RESEARCH BRIEF

**Status: RESEARCH / DESIGN ONLY.** No provider is selected, no catalog is built, no audio
infrastructure is implemented. This brief scopes the open questions M-d must answer *before* any
implementation milestone is approved. It is the handoff target of the frozen `music_plan/1` contract
(M-b) and the precursor to the Edit Planner milestone.

Sequencing (from the Music reconciliation): **M-a reconciliation ✓ → M-b MusicPlan intelligence FROZEN ✓
→ M-c MusicDirection infer + Figma MUSIC line (no audio) → M-d (this) → Edit Planner (separate).**

---

## 0. Why this brief exists

M-b froze the Story-level **soundtrack intelligence** (`MusicPlan`): character, scope-sensitive
energy_arc, `audio_strategy`, optional `track_selection_criteria`. It selects *nothing* — it describes
what the soundtrack should DO. M-d is the deliberately-separated other half: **where an actual, legally
usable audio track comes from, how we analyse its real structure, and how that structure reaches the
Edit Planner.** Keeping these apart is the whole point of the A/B split — intelligence froze without
taking on catalog/licensing/delivery risk.

**Hard rule for M-d research:** a *streaming* license (Spotify, Apple Music, YouTube Music) does NOT
grant *synchronization* or *export/distribution* rights. Embedding music in an exported/shareable video
is a sync-license problem, not a playback problem. Do not conflate the two.

---

## 1. Export-safe music licensing (the gating question)

The MVP produces a video the user **exports and shares**. That requires, at minimum, a **synchronization
license** (music timed to picture) and a **master-use license** (the specific recording), OR a single
license that bundles both for UGC export. Research must resolve, per candidate path:

| Path | Sync+master for export? | Per-video royalty/attribution? | Catalog breadth | Integration cost |
|---|---|---|---|---|
| **Royalty-free / buyout catalogs** (Epidemic Sound, Artlist, Musicbed, Soundstripe, Uppbeat) | Typically YES for social/UGC export under subscription | Subscription-based; attribution varies | Large, curated, mood-tagged | API/catalog access varies — must confirm programmatic access tier |
| **Production-music / PRO-cleared libraries** | YES but often per-use cue-sheet reporting | Per-use or blanket | Large | Heavier reporting obligation |
| **Creative-Commons / CC0** (e.g. some Free Music Archive, ccMixter) | YES if license permits commercial + derivative | Attribution often required (CC-BY) | Variable quality/breadth | Cheapest; license-hygiene burden on us |
| **AI music generation** (e.g. generated-track providers) | Depends on provider ToS granting export/commercial rights | Per-track or subscription | Infinite but quality/consistency risk | Must confirm the ToS actually assigns usable rights |
| **User-imported music** | **NO** — user rarely holds sync rights to commercial tracks | N/A | User's own library | Lowest infra, highest legal risk to the USER and platform |
| **Streaming APIs** (Spotify/Apple/YouTube) | **NO sync/export rights** | — | — | **Rejected for export** — playback-only |

**Open decisions for the owner (M-d):**
1. Does the MVP ship a **small curated licensed catalog** (simplest to clear, matches `story_catalog`
   source already in `MusicAssetContract`), or defer music from the exported MVP entirely?
2. Is **user-imported music** allowed, and if so with what explicit in-app warning that the user is
   responsible for their own rights (and that we do not clear sync for arbitrary imports)?
3. Attribution/royalty model acceptable for the business?

**Recommendation to evaluate (not decided):** start with a **small, buyout/royalty-free curated catalog**
(≈20–50 tracks spanning the mood/energy space the frozen `character.mood_tags`/`energy` vocabulary uses).
It is the cheapest path to a legally clean export and maps 1:1 onto the existing `source="story_catalog"`.

---

## 2. Catalog / API access

If a catalog path is chosen, research must determine:
- Programmatic access: does the provider expose a **search/download API** keyed on mood/energy/BPM, or
  only a manual web library? (`track_selection_criteria` → catalog query needs structured search.)
- Licensing of **cached/derived** copies on our infra vs. fetch-on-demand.
- Whether track **analysis (BPM/beats/sections)** is provided by the catalog or must be computed by us
  (see §3). Several royalty-free providers ship BPM + "energy" tags but not beat grids / downbeats.
- Delivery: do we store masters in our own S3/CDN (a real infra decision — deferred), or stream from the
  provider at render time?

`MusicAssetContract` already models this: `{id, title, duration_seconds, mood_tags, source, analysis}`.
M-d decides how `source="story_catalog"` is populated and whether `analysis` comes from the provider or us.

---

## 3. Local audio analysis (beats / downbeats / sections / energy / vocals)

The frozen `MusicAnalysisContract` today has `{bpm, beats[], sections[]}`. M-d must decide which analysis
dimensions the Edit Planner actually needs, and where analysis runs. Candidate dimensions and their
Edit-Planner consumers:

| Analysis output | Edit Planner use | Build vs. buy |
|---|---|---|
| **BPM** | tempo-aware shot pacing | catalog-provided or trivial to compute |
| **Beat timestamps** | beat-synced cuts | librosa/aubio-class detection, or catalog |
| **Downbeats / bar structure** | land major cuts on bar boundaries, not just beats | harder; needs a downbeat model |
| **Sections** (intro/build/drop/outro) | align `energy_arc` regions to the track's real structure | section/segmentation model |
| **Energy curve over time** | match the track's actual dynamics to `energy_arc` intensities | RMS/loudness envelope — cheap |
| **Vocal regions** | duck music under native dialogue/singing, keep instrumental passages for talk | source-separation / vocal-activity detection — moderate |

**Where it runs:** on-device vs. server vs. precomputed-per-catalog-track. **Recommendation to evaluate:**
if the MVP uses a curated catalog, **precompute analysis once per catalog track server-side** (no live
analysis cost, deterministic, cacheable). User-imported music would need on-demand analysis — another
reason to gate or defer imports.

**Downbeats, energy curves, vocal regions are explicitly future per the architecture** — M-d decides which
graduate into the frozen `MusicAnalysisContract` and which stay out of the MVP.

---

## 4. How actual track structure is consumed by Edit Planner (the seam)

This is the load-bearing interface question. Two distinct structural inputs reach the Edit Planner:

1. **MusicPlan (frozen, M-b)** — the *intended* soundtrack shape: `energy_arc` in **normalized fractions**
   (0..1), categorical intensity, `audio_strategy`. This is intent, independent of any specific track.
2. **MusicAnalysis (M-d)** — the *actual* track's structure: real seconds, real beats, real sections.

The Edit Planner **reconciles the two**: it maps MusicPlan's fractional energy regions onto the chosen
track's real sections/beats, then makes the concrete cut decisions. Crucially:

- **MusicPlan never has cut points or seconds** — it stays fractional/categorical (M-b freeze).
- **MusicAnalysis has real seconds/beats** — but makes NO creative decisions.
- **Edit Planner owns the actual cuts, transitions, timing, and audio placement** — consuming both plus
  CreativeIntent + StoryGuide + Coverage/2 + MomentUnderstanding[]. Music must not become the Edit Planner.
- `audio_strategy` (soundtrack_role / native_audio_priority) tells the Edit Planner *when the composed
  track must recede* for native audio (the real guitar performance, birthday singing, dialogue). The exact
  ducking/fade/mix is the Edit Planner's / audio-mix's job, driven by this Story-level signal.

**Open design question for M-d/Edit Planner:** when the chosen track's real section count/placement differs
from MusicPlan's fractional arc (it usually will), who wins? Proposed principle to validate: **MusicPlan is
the creative intent; MusicAnalysis is the physical constraint; the Edit Planner satisfies intent *within*
the track's real structure** — e.g. it places the "payoff" region near the track's real climax section
rather than mechanically at fraction 0.75.

---

## 5. Explicitly OUT of scope for M-d research (do not do during research)
- Selecting a provider.
- Building a catalog or ingesting tracks.
- Standing up S3/CDN/audio delivery infrastructure.
- Implementing MusicAnalysis.
- Any Edit Planner implementation.

## 6. M-d research deliverables (when M-d is actually run)
1. Export-safe licensing recommendation (which path, with the sync/master-rights analysis).
2. Catalog/provider shortlist with programmatic-access + rights confirmation.
3. Which analysis dimensions enter the frozen `MusicAnalysisContract` (and which stay future).
4. Where analysis runs (precompute-per-catalog-track vs on-demand).
5. The MusicPlan↔MusicAnalysis↔EditPlanner reconciliation contract (the §4 seam), as a design proposal.
6. User-imported-music decision + the rights-responsibility UX if allowed.
7. Infrastructure implications + cost, as an owner decision — NOT built during research.

---

## 7. Dependencies & status
- **Upstream (ready):** `music_plan/1` FROZEN (M-b); `MusicDirectionContract`, `MusicAssetContract`,
  `MusicAnalysisContract` exist in `contracts/schemas/ai_contracts.py` (MusicAsset/Analysis are
  Milestone-I placeholders, not yet exercised).
- **Blocks:** Edit Planner (needs the §4 seam decided); any music in the exported MVP.
- **Not blocked by M-d:** M-c (MusicDirection infer + Figma MUSIC line) — pure intelligence, no audio.

**M-d remains a research brief until the owner approves running it. No provider, no infra, no implementation.**
