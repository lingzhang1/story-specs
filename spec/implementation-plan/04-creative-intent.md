# 03 — Creative Intent

## Overview

Accepts user input (style template, natural language, optional reference media) and normalizes it into a structured `CreativeIntent` object. First AI layer in the pipeline.

---

## Requirements

- Three input types: style template, natural language description, reference media (video/photo/URL) — all optional individually; at least one required
- Output is typed `CreativeIntent` JSON — no prose passed downstream
- For reference media: extract energy, pacing, shot length, camera motion, subject/environment balance — extract transferable creative grammar only, do not copy content or location
- User can optionally specify target duration; defaults to 10 seconds if omitted
- No filmmaking vocabulary in UI or prompts (no "B-roll", "dolly", "rack focus")
- Latency: < 3 seconds for text/template, < 8 seconds for reference video

---

## Design

**Input screen:**
```
Template picker (visual grid, not text labels)
  +
Text field: "Describe what you want (optional)"
  +
Reference: "Add a video or photo for style (optional)"
  ↓
[Continue]
```

**Processing — text + template path:** Single LLM call. Output: `CreativeIntent` JSON.

**Processing — reference path:**
1. Extract 5 representative frames (AVFoundation)
2. VLM call: analyze frames for energy, pacing, camera motion, subject/environment balance
3. Merge with template/text intent

**Rule:** Text overrides template defaults. Template provides defaults when text is absent.

**Style templates (MVP — manually curated):**

| ID | Display name | Energy | Pacing | Camera motion |
|---|---|---|---|---|
| cinematic | Cinematic | low–medium | slow | smooth |
| energetic | Energetic | high | fast | dynamic |
| fashion | Fashion | medium | medium | smooth |
| romantic | Romantic | low | slow | smooth |
| calm | Calm / Aesthetic | low | slow | static |
| adventure | Adventure | high | medium–fast | dynamic |
| playful | Playful | high | fast | dynamic |
| main_character | Main Character | medium–high | medium | follow |
| landscape | Landscape | low–medium | slow | reveal |
| social_native | Social Native | high | fast | dynamic |

**Edge cases:**
- No text + no reference → template defaults are the full intent
- Conflicting signals (calm template + "I want it really fast") → text overrides
- Reference extraction fails → fall back to template + text, notify user

---

## Tasks

**Phase 1 (fake):**
- [ ] Build template picker UI with visual thumbnails
- [ ] Build text input field
- [ ] Build reference media picker (photo library + video)
- [ ] Hard-code `CreativeIntent` output based on selected template

**Phase 2 (real AI):**
- [ ] Design and test LLM prompt for text → `CreativeIntent` JSON
- [ ] Implement reference frame extraction (AVFoundation)
- [ ] Design and test VLM prompt for reference → `CreativeIntent` fields
- [ ] Implement merge logic (template defaults + text overrides + reference)
- [ ] Measure latency; optimize if > 3s text / > 8s reference
- [ ] Evaluate output quality across 20 varied inputs
- [ ] Build test harness: input → JSON → human review
