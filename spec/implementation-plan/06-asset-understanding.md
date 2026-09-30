# 06 — Asset Understanding

## Overview

Analyzes existing or newly captured clips to produce per-asset metadata. Used by Edit Planner and Story Reconciliation to understand what material is actually available and what story is achievable.

---

## Requirements

- Per-asset structured JSON output — no prose
- Detect usable time range per clip (trim shaky setup, obstructions, camera drops)
- Assign potential `story_roles` each clip could fill
- Rate quality: unusable / poor / acceptable / good
- Produce one-sentence semantic description of what's happening in the clip
- Handle imperfect footage — shaky, mixed orientations, partial subjects are valid input; characterize what's there, don't fail on imperfection
- Process clips in parallel when multiple are provided
- Latency: < 5 seconds per clip

---

## Design

**Pipeline per clip:**
```
Video clip
        ↓
Frame sampler: 5 frames (start, 25%, 50%, 75%, end) — AVAssetImageGenerator, resize 768px
  +
Basic motion energy heuristic (AVFoundation)
        ↓
VLM call
        ↓
Asset metadata JSON
```

**Usable range detection:**
- Motion energy heuristic: high motion at start/end often = setup/drop
- VLM frame analysis: identify frames where subject is blocked or footage is degraded
- Heuristic default: trim first 0.5s and last 0.3s unless clip < 2s

**Quality rubric:**

| Rating | Criteria |
|---|---|
| good | Stable, subject clear, usable range > 60% |
| acceptable | Minor issues, usable range > 30% |
| poor | Significant issues, some usable frames |
| unusable | Completely unstable, subject never visible, < 0.5s usable |

**Story roles (fixed list — VLM selects all applicable):**
- `establishing` — introduces the environment
- `movement` — subject or camera in motion
- `reveal` — something being revealed
- `human_moment` — candid or emotional human content
- `landscape` — environment without subject
- `detail` — close-up of object or surface
- `transition` — useful as a transition element
- `climax` — visually strongest moment

**Output example:**
```json
{
  "id": "asset_03",
  "duration": 8.4,
  "usable_range": { "start": 1.1, "end": 6.7 },
  "semantic_description": "Person walking through a crowded street toward illuminated signs at night.",
  "quality": "good",
  "story_roles": ["movement", "establishing"],
  "scene": "urban street, night",
  "subjects": ["one person walking away from camera"],
  "camera_motion": "follow",
  "composition": "medium-wide"
}
```

---

## Tasks

**Phase 1 (fake):**
- [ ] Accept clip selection (up to 10 clips)
- [ ] Hard-code metadata: all clips "acceptable", usable_range = full clip
- [ ] Assign story roles by clip order (first = establishing, etc.)

**Phase 6 (real AI):**
- [ ] Implement frame sampler (5 frames, resize 768px)
- [ ] Implement motion energy heuristic for usable range detection
- [ ] Design and test VLM prompt → asset metadata JSON
- [ ] Implement parallel processing for multiple clips
- [ ] Measure per-clip latency; optimize if > 5s
- [ ] Test with 20 diverse clips including poor/random footage
- [ ] Validate usable_range accuracy against manual ground truth labels
