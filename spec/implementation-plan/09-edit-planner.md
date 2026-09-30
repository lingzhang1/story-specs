# 09 — Edit Planner

## Overview

Takes `CreativeIntent` + understood assets and produces a complete `EditPlan`. This is Story Reconciliation — finding the best achievable story from what actually exists, not forcing assets into a predetermined structure.

---

## Requirements

- Output: typed `EditPlan` JSON — no prose. Media engine consumes this directly.
- Works for both paths:
  - **DIRECT:** original `StoryPlan` + captured assets → reconcile → `EditPlan`
  - **EDIT:** `CreativeIntent` + existing assets → discover achievable story → `EditPlan`
- Story Reconciliation: if actual assets tell a stronger narrative than the original plan, use that narrative instead
- Handles imperfect assets: produce an `EditPlan` even if some shots failed or some clips are poor quality
- Only fails if no usable material exists at all
- Every segment references `asset_id`, `start`, `end`, `speed` — no ambiguity for media engine
- Output duration within ±20% of `creative_intent.desired_duration`
- `EditPlan` includes `music_track_id` and `beat_sync` flag

---

## Design

**Pipeline:**
```
CreativeIntent
  +
AssetUnderstanding[] (all available assets with usable_ranges)
  +
StoryPlan (DIRECT only; null for EDIT)
        ↓
LLM call (reconciliation mode)
        ↓
EditPlan JSON
```

**DIRECT path prompt:** "Here is the original shot plan and the clips that were actually captured. Some shots may have failed. Produce an EditPlan that achieves the intended story as closely as possible using the available clips."

**EDIT path prompt:** "Here are the available clips and the user's creative intent. Discover the strongest story achievable from this material. You are not bound to any pre-existing plan."

**EditPlan example:**
```json
{
  "total_duration": 9.2,
  "segments": [
    {
      "asset_id": "asset_03",
      "start": 1.1,
      "end": 4.8,
      "speed": 1.0,
      "crop": null,
      "transition_in": "cut"
    },
    {
      "asset_id": "asset_07",
      "start": 0.5,
      "end": 3.9,
      "speed": 0.85,
      "crop": { "x": 0, "y": 0.1, "width": 1.0, "height": 0.8 },
      "transition_in": "crossfade"
    },
    {
      "asset_id": "asset_04",
      "start": 5.0,
      "end": 8.4,
      "speed": 1.0,
      "crop": null,
      "transition_in": "cut"
    }
  ],
  "music_track_id": "track_cinematic_01",
  "beat_sync": true
}
```

**Validation before passing to media engine:**
- All `asset_id` values exist in project
- All `start`/`end` within asset `usable_range`
- `total_duration` matches sum of segment durations ± rounding

**Hard boundary:** AI generates the EditPlan. Media engine executes it. Never ask an LLM to "edit the video."

---

## Tasks

**Phase 1 (fake):**
- [ ] Hard-code EditPlan: clips in order, full usable_range, default music
- [ ] Validate hard-coded plan passes schema checks

**Phase 8 (real AI):**
- [ ] Design DIRECT reconciliation prompt → EditPlan JSON
- [ ] Design EDIT discovery prompt → EditPlan JSON
- [ ] Implement EditPlan schema validation
- [ ] Test DIRECT path: original plan vs. what was actually captured (including failed shots)
- [ ] Test EDIT path: random unplanned clips → achievable story
- [ ] Verify output duration accuracy (within ±20% of target)
- [ ] Handle edge case: < 2 usable clips (return error, not crash)
- [ ] Test Story Reconciliation: verify it surfaces stronger narratives when available
