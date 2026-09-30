# 05 — Story Director

## Overview

The most important AI component in the MVP. Takes `CreativeIntent` + `SceneUnderstanding` and produces one recommended `StoryConcept` + `StoryPlan`. This is what determines whether Story creates real value.

---

## Requirements

- Input: `CreativeIntent` + `SceneUnderstanding` (nullable). Output: `StoryConcept` + `StoryPlan`. Structured JSON only.
- Recommend ONE concept by default — do not force the user to evaluate three options
- Concepts must be scene-specific — reference actual elements from `SceneUnderstanding`. A generic "walk → turn → reveal" that works anywhere is a failure.
- Optimize for low capture effort: max 3 shots (preferred 2), max ~30 seconds total capture time, minimize repositioning
- Every suggested action must be physically feasible given `constraints` in `SceneUnderstanding`
- `guidance_instructions[]` must be fully pre-computed at plan generation time — no AI calls during real-time capture
- Latency: target < 5 seconds, max 8 seconds
- Output must be consistent with `CreativeIntent` style (energy, pacing, subject/environment balance)

---

## Design

**Pipeline:**
```
CreativeIntent + SceneUnderstanding (nullable)
        ↓
Story Director LLM call
        ↓
StoryConcept + StoryPlan JSON
```

**Prompt structure:**
1. Role: creative director helping someone make a short compelling video
2. Hard constraints: max shots, max capture time, physical feasibility rules
3. Output: strict JSON schema
4. Creative grammar library (curated named patterns)
5. Anti-patterns: generic sequences that ignore the scene

**Creative grammar library (MVP starter set — manually curated):**

| Name | Description |
|---|---|
| Follow | Camera follows subject from behind |
| Reveal | Subject moves to expose environment |
| Push-in | Camera moves toward subject or scene element |
| Foreground wipe | Subject passes close to camera, creating transition |
| Static subject / moving environment | Subject still, world moves around them |
| POV | Camera takes subject's point of view |
| Human-vs-landscape | Wide shot showing scale relationship |
| Entrance | Subject enters frame from edge |
| Detail → wide | Begin on detail, pull back to reveal scene |

Expand this library based on evaluation dataset failures.

**StoryPlan output example:**
```json
{
  "concept": {
    "title": "Main Character",
    "description": "Move through the crowd before the city reveals itself",
    "duration": 9
  },
  "shots": [
    {
      "id": "shot_01",
      "order": 1,
      "purpose": "establish movement and energy",
      "subject_action": "walk forward through the crowd",
      "camera_action": "follow from behind at medium distance",
      "framing": "medium-wide",
      "duration": 4,
      "guidance_instructions": [
        "Walk toward the signs.",
        "Ready?",
        "3… 2… 1… go."
      ],
      "status": "pending"
    },
    {
      "id": "shot_02",
      "order": 2,
      "purpose": "environment reveal",
      "subject_action": "stop and look up at the signs",
      "camera_action": "step slightly to the side",
      "framing": "wide",
      "duration": 5,
      "guidance_instructions": [
        "Stop here.",
        "Look up at the signs.",
        "Hold still.",
        "3… 2… 1… go."
      ],
      "status": "pending"
    }
  ]
}
```

**Evaluation dimensions (must score > 80% before Alpha 0.1):**
1. Scene-specific — references actual scene elements
2. Physically possible — respects constraints
3. Would I actually shoot this — real-world plausibility
4. Varied — different from other suggestions for similar scenes
5. Style match — consistent with requested CreativeIntent

See spec `13-evaluation` for full dataset and process.

---

## Tasks

**Phase 1 (fake):**
- [ ] Hard-code 3 concept + plan pairs (cinematic, energetic, romantic)
- [ ] Return appropriate hard-coded plan based on selected template

**Phase 4 (real AI):**
- [ ] Design system prompt (role, constraints, output format, anti-patterns)
- [ ] Curate creative grammar library (10 named patterns minimum)
- [ ] Implement Story Director API endpoint
- [ ] Test against 30 evaluation scenarios (spec 13)
- [ ] Iterate on prompt until evaluation score > 80% across all dimensions
- [ ] Measure latency; optimize if > 5s
- [ ] Handle null `SceneUnderstanding` (intent-only path)

**Ongoing:**
- [ ] Expand creative grammar library based on field test failures
- [ ] Track variety — detect if Director falls back to generic patterns
- [ ] Add evaluation scenarios as new scene types are tested
