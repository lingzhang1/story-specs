# Milestone M-d — Music Sourcing & Analysis: RESEARCH BRIEF (full)

**Status: RESEARCH / DESIGN ONLY — STOP for owner review.** No provider selected, no catalog built, no
audio infrastructure, no MusicAnalysis/Edit Planner/Media Engine implementation, no licenses purchased,
no vendors contacted. All pricing/rights below are from public pages (dated where known) and are RESEARCH
CANDIDATES, not approved vendors or quotes. Anything not publicly verifiable is flagged for commercial/
legal confirmation.

Pipeline this unblocks: **MusicPlan (frozen `music_plan/1`) → actual track → MusicAnalysis → Edit Planner
→ rendered export.**

---

## 1. The gating rule (unchanged from the stub, restated)
A **streaming** license (Spotify/Apple/YouTube Music) does NOT grant **synchronization** (music timed to
picture) or **export/distribution** rights. Embedding music in an exported, shareable video needs a
**sync + master-use** license, OR a single UGC/sublicense that bundles both. **A creator subscription that
lets the SUBSCRIBER publish videos is NOT automatically a license letting Story's USERS generate and export
videos** — that distinction (per-subscriber vs. platform-sublicensing-to-end-users) is the whole provider
decision.

---

## 2. Music sourcing landscape — three strategies

### A. Licensed catalog provider (verified research)

| Provider | API / programmatic | End-user **sublicensing** (the key test) | Sync for user-exported video | Model | Verified? |
|---|---|---|---|---|---|
| **Epidemic Sound — Partner API** | **Yes** — self-serve dev portal, free prototyping tier, full catalog (~55k tracks), mood/theme playlists | **YES — "Sublicensing for users is supported"** (explicitly built for platforms whose end users soundtrack content) | Yes, "all rights included", cross-platform; certified TikTok Sound Partner | Partner/enterprise (pricing not public → sales) | API + sublicensing model VERIFIED from public pages; **pricing UNKNOWN → commercial confirmation required** |
| **Artlist** | Catalog + Clearlist; API is enterprise | Standard license is **per-subscriber, per-channel** (up to 3 channels/platform) — NOT a sublicense to arbitrary app end-users | Yes for the subscriber's own projects | Subscription | License model VERIFIED (per-channel subscriber); **end-user sublicensing NOT offered on public tiers → fits Story-owned-catalog, not sublicensing** |
| **Jamendo Licensing** | **Yes** — API free for non-commercial; **commercial = contact for quote** | Per-track **perpetual sync** licenses, incl. "mobile apps / video games" as named project types | Yes — Standard: one track across unlimited videos of ONE channel; per-project perpetual | Per-track (~from a few $) or subscription (~$29/mo) | Per-track sync + app-project licensing VERIFIED; **commercial API terms = quote required** |
| **Soundstripe / Shutterstock Music / Uppbeat / Musicbed** | Varies (mostly catalog, limited public API) | Mostly per-subscriber creator licenses | Yes for subscriber | Subscription | Not deep-researched; **secondary candidates** |

**Takeaway:** only **Epidemic Sound's Partner API** publicly advertises the exact thing a third-party app
like Story needs — **sublicensing the music to OUR end users** so each user's exported video is cleared —
with a real search/download API. That makes it the natural **Strategy-A** candidate, with the single open
item being **partner pricing (sales-gated, not public)**.

### B. Story-owned small licensed catalog (≈30–100 tracks)
Acquire a curated set once, under licenses that permit **redistribution inside rendered user videos**, and
host them ourselves. Fits the existing `MusicAssetContract.source = "story_catalog"`.
- **Rights path:** per-track perpetual sync from Jamendo (named "mobile app" use), direct composer licensing,
  or a buyout library — must explicitly grant **our users' exported videos**, worldwide, surviving
  subscription end (perpetual/buyout preferred so a lapsed subscription doesn't orphan already-rendered
  videos).
- **Cost:** one-time per-track (Jamendo per-track, or composer flat fee) + storage. ~30–100 tracks is enough
  to span the frozen `character.mood_tags`/`energy` space for **dogfooding**.
- **Pros:** full control, deterministic, analysable once offline, cheapest at small scale, no per-export fee.
  **Cons:** limited diversity, curation + metadata upkeep, we hold the compliance burden.
- **Assessment: a small high-quality catalog is SUFFICIENT for MVP dogfooding** and is the lowest-risk way to
  get the whole pipeline working end-to-end before committing to a provider contract.

### C. User-imported music
Import from the iOS Files app (`UIDocumentPicker` / `.audio` UTType; m4a/mp3/wav/aac/caf), analyse locally
or on upload, cache.
- **Rights:** owning a file ≠ holding sync rights. A user importing a commercial track CANNOT legally export
  it in a shared video. So this path is **personal/unlisted use only, with an explicit in-app notice that
  the user is responsible for their own rights**, OR gated out of the shareable-export path.
- **Pros:** zero licensing cost/infra to us, user autonomy. **Cons:** highest legal ambiguity, per-file
  on-demand analysis, format/decoding variance.
- **Assessment: allow as a SECONDARY path with a clear rights disclaimer; do NOT make it the MVP default.**

---

## 3. Economics (published where available; else labelled)

| Cost axis | Story-owned catalog (B) | Epidemic Partner API (A) | User import (C) |
|---|---|---|---|
| Licensing | one-time per-track (Jamendo per-track ~$, or composer flat); **perpetual** | **UNKNOWN — partner pricing is sales-gated** (likely per-MAU or rev-share) | $0 (user's responsibility) |
| API fees | none (we host) | included in partner deal (unknown) | none |
| Audio delivery | existing S3 (small) | provider-streamed or cached | device-local |
| Storage | ~tens–hundreds of MB (30–100 tracks) in existing S3 | cache only | device |
| Analysis | **once per track, offline** (fixed, tiny) | once per track on ingest | per import, on-demand |
| Recurring fixed | ~$0 beyond S3 | partner subscription (unknown) | $0 |
| Per-video | **$0** | possibly per-export/per-MAU (unknown) | $0 |

Scale estimates — because the catalog path has **no per-export licensing fee**, cost is ~flat regardless of
users/exports (dogfooding → 10k users → still just storage + one-time track licensing). **Epidemic cannot be
costed without the partner quote** — flagged. User-import is ~$0 to us at every scale. *(All catalog numbers
are order-of-magnitude; Jamendo per-track and the $29/mo subscription are the only published anchors. No
enterprise quote is fabricated.)*

---

## 4. MusicAnalysis — contract gaps & proposed minimum

Frozen today: `MusicAnalysisContract { bpm?, beats[], sections[] }`, `MusicSection { type, start_seconds,
end_seconds }`. Per-field "what Edit Planner decision improves because this exists?":

| Field | Edit Planner decision it enables | Verdict for MVP |
|---|---|---|
| `duration_seconds` | fit MusicPlan fractions → real seconds; pick a track that covers target length | **ADD** (currently only on MusicAsset; Analysis should carry it too) |
| `bpm` | tempo-aware shot pacing / cut cadence | **KEEP** |
| `beats[]` | beat-synced cuts | **KEEP** |
| `sections[]` (+types) | map `energy_arc` regions → real intro/build/peak/outro; place payoff at the real climax | **KEEP** |
| **downbeats[]** | land MAJOR cuts on bar boundaries, not just any beat | **ADD (behind an experiment)** — meaningfully better cuts, but needs a downbeat model |
| **energy_curve** (coarse samples) | match `energy_arc` intensities to the track's actual dynamics | **ADD (coarse, e.g. 1 value/sec or per-section)** — cheap (RMS), directly serves energy_arc matching |
| **vocal_regions[]** | duck/avoid music vocals under native dialogue/singing (M-b native-audio) | **ADD (behind an experiment)** — load-bearing for the native-audio promise |
| intro/outro character, "major transitions", build/drop regions | arguably derivable from sections+energy | **DEFER** — don't add until a concrete planner need appears |

**Proposed minimum `music_analysis/1`:** `{ duration_seconds, bpm?, beats[], downbeats[]?, sections[]
(type,start,end), energy_curve[]? (coarse), vocal_regions[]? (start,end) }`. Everything optional except
duration — a track with only duration+beats is still usable; richer fields unlock better cuts. **Do not add a
field the Edit Planner can't act on.** Freeze only after the §5 experiment shows which optional fields are
reliably extractable.

---

## 5. Audio-analysis technology comparison + experiment

| Tool | Beats | Downbeats | Sections | Energy | Vocals | Notes / license |
|---|---|---|---|---|---|---|
| **librosa** | good | weak | basic (novelty) | **easy (RMS)** | no | ISC, trivial install — the baseline |
| **madmom** | **excellent** | **excellent (DBN)** | some | — | — | BSD-ish but **hard to install on modern Python**; reimpls (`madmom-infer`, `BeatNet`) exist |
| **Essentia** | good | ok | good | good | some | AGPL (⚠ licensing care for a hosted service) — flag |
| **"all-in-one" music models** (HF) | good | good | **functional labels (intro/verse/chorus/outro)** | — | — | pretrained; heavier compute |
| **Demucs / source separation** | — | — | — | — | **vocal/instrumental split → vocal regions** | MIT; heavier, GPU-friendly |
| **Apple (AVFoundation/MusicKit)** | limited | no | no | partial | no | on-device only, not a structure analyser |
| **Commercial analysis APIs** | varies | varies | varies | varies | varies | adds a dependency + cost; only if build path fails |

**Build-vs-buy:** librosa covers duration/bpm/beats/energy_curve immediately and cheaply; downbeats need a
madmom-class model; sections need novelty/all-in-one; vocal regions need source-separation or a VAD-on-vocals
model. **No single library does all five well** — the pipeline is a small ensemble.

**Proposed bounded experiment (do NOT run yet; owner-gated):** 8–10 **legally usable** tracks (a few CC0/
owned, so we never analyse unlicensed audio) spanning: warm acoustic, cinematic gradual build, energetic
short-form, sparse ambient, emotional instrumental, **vocal-heavy**, clear-sections, weak/ambiguous-beat.
Run librosa (baseline) vs madmom-class (downbeats) vs an all-in-one (sections) vs a separation model (vocals).
Score: beat accuracy, section accuracy, energy-curve usefulness, vocal-region accuracy, latency, compute.
**Human review decides whether the resulting MusicAnalysis + match actually improves a Story** — not F-scores
alone. Freeze `music_analysis/1` only after this.

---

## 6. MusicPlan → track matching (simplest validatable architecture)

```
MusicPlan (music_plan/1)
  -> catalog filter      (character.mood_tags, energy, target_duration, audio_strategy)
  -> structural score    (energy_arc regions vs track sections/energy_curve; build/payoff alignment)
  -> vocal/native guard   (if native_audio_priority=important or dialogue -> penalise vocal tracks)
  -> ranked tracks -> top MusicAsset + its MusicAnalysis
```
**Deterministic, rule-based scoring — NO embeddings, vector DB, or extra LLM.** For a 30–100-track catalog a
linear scan with a weighted score is instant and fully explainable (and testable with fixtures). Weights:
mood/energy match, duration fit, arc↔section alignment, ending suitability, and a **vocal penalty** driven by
`audio_strategy`. Revisit embeddings only if a provider catalog (tens of thousands of tracks) is adopted —
and even then filter-then-score first.

---

## 7. Native audio stays first-class (preserve M-b)
`audio_strategy.{soundtrack_role, native_audio_priority}` from MusicPlan drives matching AND downstream mix:
- **Guitar:** `native_audio_priority=important` → prefer sparse/instrumental tracks, reserve the payoff for
  the real performance.
- **Birthday:** singing is the moment → duck/retreat music under it.
- **TMB:** wind/footsteps/laughter enrich → supportive, not wall-to-wall.
- **Dialogue-heavy:** penalise vocal tracks (vocal_regions) to avoid competing with speech.
MusicAnalysis **informs** these; it does **not** perform fades/ducking — that's Edit Planner / audio-mix.

---

## 8. Edit Planner boundary (final interface)
- **MusicPlan** — desired musical shape (fractions, categorical). Owns intent.
- **MusicAnalysis** — observed track structure (real seconds/beats/sections). Owns facts, no decisions.
- **MomentUnderstanding** — what each captured Moment contains.
- **Edit Planner** — OWNS footage selection, clip ordering, exact cut times, beat alignment, music-segment
  selection, soundtrack placement, native-audio preservation, transitions. Reconciles MusicPlan intent
  **within** MusicAnalysis's real structure (payoff lands at the track's real climax, not mechanically at
  fraction 0.75).
- **Media Engine** — rendering only.
None of these responsibilities move into MusicPlan.

---

## 9. Infrastructure implications (existing AWS is sufficient for MVP)
Existing: App Runner + ECR + S3 (temporary media) + Secrets Manager; SQS/Lambda/Fargate are "add only when a
workload needs it" (04-aws-infrastructure.md). For a **Story-owned catalog**: analyse each track **once,
offline, at ingest** (a local/CI script, not a service), persist `MusicAnalysis` JSON + the audio in the
existing S3 bucket, index metadata in the existing store, match in-process at Create My Story. **No new
S3/CDN/queues/workers required for MVP scale.** User-import analysis would run on upload (bounded; SQS/Lambda
only if it exceeds the 120s inline budget). **No infrastructure changes are authorised in M-d.**

---

## 10. Figma Music UX reconciliation
Frozen Figma already has the full future surface: `ScreenTMBMusic` (recommended tracks + preview), `MusicPick`
(Story's pick + Change on the Create screen), and the "Change music" affordance on the Result screen. M-c
shipped the lightweight MusicDirection (feel) line; those **track** surfaces are the M-d/post-M-d targets and
stay disabled until a source + analysis + Edit Planner exist. Minimal future flow:
`MusicDirection → suggested track (from matching) → preview → optional Change → Create My Story`.
**The music choice need NOT be finalised during Story creation** — a strong default from matching, changeable
before or after the first rendered preview, fits the "generated at/just-before Create My Story" timing and
avoids over-committing the user early. Preserve the lightweight MusicDirection UX; **do not build a music
editor.**

---

## 11. Unresolved legal / commercial questions (owner / counsel)
1. **Epidemic Partner API pricing + the exact sublicensing terms in a signed agreement** (public page says
   sublicensing supported; the contract must confirm per-MAU/rev-share and that lapse doesn't revoke
   already-exported videos).
2. For a **Story-owned catalog**: confirm each track's license explicitly covers **our end users' exported,
   shareable videos**, worldwide, **perpetual/surviving termination**.
3. **User-import**: the rights-responsibility disclaimer wording, and whether imported commercial tracks are
   blocked from the shareable-export path.
4. **Essentia's AGPL** vs. a hosted service — avoid or isolate.
5. PRO/royalty reporting obligations for whichever path (cue sheets?).

---

## 12. Recommendation (conditional)
- **Preferred MVP strategy: B — a small Story-owned curated catalog (~30–100 tracks)** licensed for end-user
  exported video (Jamendo per-track perpetual and/or direct/buyout), analysed once offline. Lowest legal and
  infra risk, no per-export fee, deterministic, gets the full pipeline working for dogfooding. **Conditional
  on per-track license verification (§11.2).**
- **Fallback / scale path: A — Epidemic Sound Partner API** (the only researched provider publicly offering
  end-user sublicensing + a real API). Adopt when catalog breadth matters, **conditional on partner pricing +
  signed sublicensing terms (§11.1).**
- **Secondary: C — user import**, personal-use/disclaimered, never the default.
- **Reject** streaming APIs (no sync/export) for the export path.
Priority order honoured: verified export rights > asset delivery > integration feasibility > quality >
matching/analysis feasibility > cost > scale. **Because licensing is not yet legally verified, this
recommendation is CONDITIONAL.**

---

## 13. Recommended M-d implementation sequence (when approved)
1. Owner picks strategy (B preferred) + legal verifies rights.
2. Acquire ~30–100 tracks; store audio + metadata in existing S3.
3. Run the §5 bounded analysis experiment on legally-usable tracks; freeze `music_analysis/1`.
4. Build the offline per-track analysis script (one-time ingest, not a service).
5. Build the deterministic matcher (§6) with fixtures.
6. Wire a suggested-track + preview into the (currently disabled) Figma track surface.
7. Hand `MusicAsset + MusicAnalysis` to the Edit Planner milestone (separate).

## 14. Decisions requiring owner approval
- Sourcing strategy (B vs A vs B-then-A) and the legal verification go-ahead.
- Whether user-import ships in MVP and its disclaimer.
- Approve running the §5 analysis experiment (and acquiring the small legal track set for it).
- Which optional MusicAnalysis fields (downbeats/energy_curve/vocal_regions) are in scope to freeze.

**STOP for owner review. No provider chosen, no license bought, no vendor contacted, no catalog/analysis/
infra/Edit-Planner built, nothing pushed.**
