# Milestone R-b Closure + R-c Create/Result Design Reconciliation

**R-b status:** Case G ACCEPTED on device (first real render). B/F now exercisable via DEBUG fixtures;
D-speed exercisable; **D-crossfade is a real engine gap (documented, not faked).** R-c = DESIGN ONLY here.
Nothing pushed. Creative Editing V2 NOT started.

Commits (app): Case G `0a1cb28`, visible entry `8236b8c`, id-case fix `9a0c353`, D/F fixtures + picker (this).

---

## 1. R-b remaining checks

### B — Native audio (device listen required)
The accepted Case G render contains **1 audio track**; probe of the exported MP4 = 720×1280, 30fps, 9.0s,
upright. The capture-audio mix applies keep=1.0 time-ranged with 0.08s ramps. **Audibility + sync is a direct
LISTENING check only** — I cannot hear the file. STOP for owner: play the exported MP4 and confirm the guitar
is audible, in sync, no discontinuity. Code path verified; perceptual check is device-only.

### D — Crossfade and speed
- **Speed (0.5x / 2.0x): exercisable now.** New DEBUG fixture `DebugEditPlanFixture.speedCases` (seg0 0.5x,
  seg1 2.0x, output_duration = span / speed). The engine implements speed via `scaleTimeRange`. Device check:
  harness → fixture → "Speed 0.5x/2x (D)" → render → confirm slow-then-fast, no black frames.
- **0.3s crossfade: NOT IMPLEMENTED (genuine R-b gap).** `MediaEngineV1.buildComposition` builds a single
  video-composition instruction with per-segment transforms and **straight cuts only** — no overlapping track,
  no opacity ramp — and the renderer `EditSegmentV2` DTO does not even decode a `transition` field. The frozen
  Opus prompt also emits `transition=none` today. So crossfade cannot be verified without ENGINE work (a second
  video track + overlap window + opacity ramp during the transition). Per the task ("do not alter the Opus
  prompt to force these effects") I did NOT fake it. **Recommendation:** implement crossfade as its own small
  verified batch (R-b.1) OR defer to R-c; it is V1-scope ("basic crossfades where supported") but was never
  built. Flagged for owner decision — NOT silently closed.

### F — Missing Moment: exercisable now.
New DEBUG fixture `DebugEditPlanFixture.missingMoment` (seg0 = unresolvable all-zero UUID, segs1-2 valid). The
engine throws `MediaEngineError.missingMoment` on the first unresolved id BEFORE export, so there is no partial
MP4 and no silent skip; source Moments are read-only (never written). Device check: harness → fixture →
"Missing Moment (F)" → expect a clear "Render failed: missingMoment(…)" status and NO result.

### R-b closure verdict
Case G (the primary goal) is **device-accepted**. B needs a 30-second listen; D-speed + F need a device tap each
(fixtures shipped); **D-crossfade needs engine work before it can pass.** I am NOT marking R-b fully CLOSED while
crossfade is unimplemented and B/D-speed/F are unrun on device — that would be claiming an unverified pass.
R-b is **substantially accepted, closure pending**: (a) owner runs B/D-speed/F, (b) owner decides crossfade now
(R-b.1) vs deferred.

## 2. First-video quality findings (honest)
- Coherent **three-shot sequence**: fretboard close → picking detail → wider performance. Narrative order tracks
  the Story Guide arc. GOOD for a first execution.
- **Simple straight-cut montage** — no transitions, no pacing sophistication. Expected: the engine does cuts.
- The **guitar-learning narrative is not fully told by 9s / 3 similar clips** — this is a SOURCE-FOOTAGE limit
  (one framing family, no "struggle→breakthrough" range), NOT a Media Engine defect. More/varied Moments would
  give the planner material to shape.
- Native-audio **quality** is unverified pending a listen.
- This is **execution proof, not finished cinematic quality.** Not claimed otherwise.

## 3. R-c Figma reconciliation (Create My Story → Result)
Frozen Figma (`prototypes/story-figma/src/App.tsx`): **Create my Story → Screen10Creating** (italic "Creating
your Story…", rotating status lines, thin progress bar; ~fixed-time simulation) **→ Screen11Result** ("Your
Story is ready.", play/preview, feedback, change-music/re-edit, implicit save/share). Music line is present but
disabled in the native build (M-c). The simulation has NO real plan/render — R-c replaces that with the real
pipeline while keeping the exact visual surface.

## 4. Current → target flow (R-c, no-music)
```
Story Home → Create my Story
  → [Creating] prepare Story intelligence (persisted CreativeIntent + completed MomentUnderstanding[])
  → (MusicPlan: SKIPPED — no production MusicPlan engine; no-music mode)
  → POST /v1/edit-plan (real Opus 4.6) → EditPlan/2
  → local validation + asset resolution (case-insensitive id → MediaStore file)
  → MediaEngineV1.render → local H.264 MP4 (RenderOutputStore)
  → persist a GeneratedOutput (real: file path + plan provenance, NOT the mock)
  → [Result] playback (AVPlayer) + Save to Photos / Share to Files
```
The DEBUG harness stays as the engineering surface; R-c is the CONSUMER surface (no diagnostics, no raw plan).

## 5. Creation lifecycle (design)
States: `idle → planning (/v1/edit-plan) → validating → rendering (progress) → done | failed | cancelled`.
- **Progress:** drive Screen10's bar from real `MediaEngineV1.render` progress (0→1), not a timer.
- **Cancellation:** harness already supports `shouldCancel`; Result-flow Cancel returns to the preview, cleans
  partial output (engine already deletes partial files on cancel/fail).
- **Failure/retry:** 503 (planner) / 422 (infeasible, show the reason) / missingMoment / export failure →
  friendly message + Retry; NEVER a fabricated montage, NEVER a partial MP4. Preserve the Story + Moments.
- **Persistence:** rendered MP4 in `<Documents>/rendered/<uuid>.mp4`; a real `GeneratedOutputModel` records the
  path + source Moment ids + plan provenance (duration, segment count, model). Survives relaunch.
- **Reopening:** Result screen for a completed Story loads the persisted output file.
- **Invalidation when Moments change:** if a Story's Moment set changes after a render, mark the output STALE
  (fingerprint of source Moment ids, like the Coverage/Guide staleness seam) and offer re-create — do not
  silently show an outdated video.
- **Duplicate-job guard:** a per-Story `isRendering` flag; a second Create tap while rendering is a no-op.

## 6. Persistence / cancellation summary
Reuse the existing `RenderOutputStore` + a new real `GeneratedOutput` (replace the Milestone-A mock). No new AWS
infra (on-device render; backend only plans). No duplicate render jobs. Source Moments never mutated/deleted.

## 7. Minimum R-c implementation scope (NOT built here)
1. Real `GeneratedOutput` persistence (file path + provenance), replacing the mock `createStoryOutput`.
2. A `StoryCreationCoordinator`: Story → `EditPlanRequestBuilder` → `/v1/edit-plan` → validate → render →
   persist, with the lifecycle states + duplicate guard + staleness fingerprint.
3. Wire Screen10Creating to real progress; wire Screen11Result to the persisted MP4 (AVPlayer + Save/Share).
4. Enable the Figma "Create my Story" button (currently a disabled `StoryUnavailableButton`) in no-music mode.
5. Failure/retry UX per §5. Keep the DEBUG harness separate and unchanged.
6. Tests: coordinator state machine, persistence/reopen, staleness, failure routing; parse-clean + XCTest.

## 8. Physical-device acceptance plan (R-c)
Create a Story (≥3 analyzed Moments) → Create my Story → see real progress → Result plays the real MP4 →
Save to Photos / Share → relaunch shows the same output → add/remove a Moment → output marked stale, re-create
works → force a planner failure (airplane mode) → friendly retry, Story intact.

## 9. Contract gaps
- **Crossfade:** EditPlan/2 CAN carry a transition (frozen contract has it), but the renderer DTO does not decode
  it and the engine does not render it. No frozen-contract change needed — this is an ENGINE gap. (R-b.1 or R-c.)
- Otherwise none: `/v1/edit-plan` + EditPlan/2 + MediaEngineV1 cover the no-music R-c flow as proven by Case G.

## 10. Recommended R-c sequence
R-b.1 (optional, small): implement + device-verify the 0.3s crossfade in the engine → closes Case D fully.
Then R-c: (1) real GeneratedOutput + coordinator → (2) wire Creating/Result screens → (3) enable Create button →
(4) failure/retry + staleness → (5) tests → (6) device gate. Production music (paid Epidemic) and Creative
Editing V2 (match cuts / motion / speed ramps / sound bridges / color / layers / split-screen / masking —
needs the R-a Edit Evidence experiment) remain SEPARATE, NOT started.

## 11. Creative Editing V2 backlog (recorded, not implemented)
match cuts · motion continuity · speed ramps · sound bridges · color matching · layered compositions ·
split screens · masking → all require the separate Edit Evidence experiment (R-a design). AU/Edit-Planner
contracts NOT reopened.
