# 04 — Scene Understanding

## Overview

Analyzes a short environmental scan video to produce a `SceneUnderstanding` object. Used by Story Director to generate scene-specific concepts rather than generic ones. DIRECT mode only; optional and skippable.

---

## Requirements

- Input: 3–5 second video of the user's environment ("show me where you are")
- Extract 5 representative frames — do not send full video to model in MVP
- Output: typed `SceneUnderstanding` JSON — no prose downstream
- Output must include `constraints` so Story Director avoids physically impossible suggestions
- Optional and skippable — if skipped, `scene_understanding` is null and Story Director falls back to intent-only reasoning
- No 3D reconstruction, no LiDAR, no depth maps in MVP
- Latency: < 5 seconds from scan completion to result

---

## Design

**Pipeline:**
```
3–5 sec scan video
        ↓
Frame sampler: 5 frames at even intervals (AVAssetImageGenerator)
Resize to 768px wide
        ↓
VLM call (frames + CreativeIntent.context)
        ↓
SceneUnderstanding JSON
```

**Scan UX:**
```
"Show me where you are"
  ↓
Camera viewfinder (rear camera)
  ↓
[Hold to scan] — records 3–5 sec
  ↓
"Got it, analyzing…"
  ↓
Result feeds silently into Story Director (user never sees raw JSON)
```

**Key schema fields to populate:**
- `scene_type` — e.g. "busy urban square", "mountain viewpoint"
- `lighting` — e.g. "night / artificial neon", "golden hour"
- `interesting_elements` — visual hooks Story Director must incorporate
- `constraints` — physical limitations (crowded, low light, narrow space)
- `possible_subject_actions` — what the subject can realistically do here
- `possible_camera_actions` — what camera movements are feasible

**Prompt principle:** Ask model to fill structured fields. Emphasize physical possibility and constraints. No narrative prose in output.

---

## Tasks

**Phase 1 (fake):**
- [ ] Build scan UI (camera viewfinder + hold-to-scan)
- [ ] Implement 3–5 sec video capture
- [ ] Hard-code `SceneUnderstanding` output (ignore actual footage)
- [ ] Show "Got it, analyzing…" → proceed to concept

**Phase 3 (real AI):**
- [ ] Implement frame sampler (AVAssetImageGenerator, 5 frames, resize 768px)
- [ ] Design and test VLM prompt → `SceneUnderstanding` JSON
- [ ] Measure end-to-end latency; must be < 5s
- [ ] Handle low-confidence or refusal outputs gracefully
- [ ] Test with 15 diverse real-world scenes
- [ ] Measure: do Story Director concepts improve when scene understanding is present vs. absent?
