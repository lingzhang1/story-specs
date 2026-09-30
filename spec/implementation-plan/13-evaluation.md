# 13 — Evaluation

## Overview

The evaluation dataset and process for measuring Story Director quality and overall MVP success. Start building this at the same time as Phase 4 (Story Director) — do not wait until later. The rated outputs are the first Story-specific proprietary dataset.

---

## Requirements

- ~30 scenarios created before Story Director is complete
- Covers diverse scenes and intents — not dominated by one scene type
- Human evaluation (no automated metrics — there is no objective measure for story quality)
- Five specific evaluation dimensions, binary pass/fail per dimension
- > 80% pass rate across all dimensions required before Alpha 0.1
- User test protocol for both DIRECT and EDIT paths
- Primary success metric: "Would you use Story next time?"

---

## Scenario Dataset

**Scene types (cover all):**

| # | Scene |
|---|---|
| 1 | Busy urban square — night |
| 2 | Busy urban square — day |
| 3 | Downtown street |
| 4 | Beach — sunset |
| 5 | Beach — midday |
| 6 | Mountain viewpoint |
| 7 | Forest trail |
| 8 | Park |
| 9 | Restaurant interior |
| 10 | Cafe |
| 11 | Hotel room |
| 12 | Museum exterior |
| 13 | Snow landscape |
| 14 | Landmark bridge / viewpoint |
| 15 | Market / food stalls |

**Intents per scene (minimum):** cinematic, energetic, romantic/calm, landscape-focused, main_character

~75 outputs to evaluate in the initial dataset.

---

## Evaluation Rubric

For each Story Director output, human rater scores 5 dimensions (pass / fail):

| # | Dimension | Pass condition |
|---|---|---|
| 1 | Scene-specific | Concept references actual scene elements; could only work here |
| 2 | Physically possible | A non-filmmaker can execute these instructions in this space |
| 3 | Would I actually shoot this | Real-world plausibility; not awkward or embarrassing to attempt |
| 4 | Varied | Meaningfully different from other outputs for similar scenes |
| 5 | Style match | Plan feels consistent with the requested CreativeIntent |

**Target before Alpha 0.1:** > 80% pass rate across all five dimensions.

---

## User Test Protocol

### DIRECT experiment

**Control:** "Make a 10-second video however you normally would."
**Experiment:** "Use Story to make a 10-second video."

Blind comparison: show both outputs to 3 independent viewers, no context. Ask: "Which is better?"

Creator survey:
- Which process would you use next time?
- Did Story suggest something you wouldn't have thought of?
- Did using Story feel like more or less work than normal?
- How likely are you to post the Story output?

### EDIT experiment

Give users 5–10 of their own random clips. Ask them to make something with Story, then without Story (their normal workflow or a template editor).

Measure: time spent, output preference (blind), perceived originality, intent match, willingness to use again.

### Target users

~20–30 people who: travel / go out / care about good content / post occasionally / don't enjoy serious editing. Not professional videographers.

---

## Primary Success Metric

> **"Would you use Story the next time you wanted to create this kind of content?"**

A beautiful demo is not enough. The workflow must win.

---

## Storage

```
docs/evaluation/
  scenarios.json       — all scene + intent combinations
  outputs/             — Story Director outputs per scenario
  ratings/             — human rating records
  results.md           — aggregate pass rates and findings
```

---

## Tasks

**Phase 4 (start immediately):**
- [ ] Create `scenarios.json` with 30 scene + intent combinations
- [ ] Set up tooling to run Story Director against all scenarios
- [ ] Establish rating process (spreadsheet or lightweight tool)
- [ ] Run Story Director against all scenarios
- [ ] Human-rate all outputs across 5 dimensions
- [ ] Calculate pass rates; iterate on Story Director prompt until > 80%

**Alpha 0.1 gate:**
- [ ] Evaluation score > 80% across all dimensions
- [ ] At least 3 team members completed DIRECT end-to-end in a real environment
- [ ] At least one "I wouldn't have thought to shoot it that way" verdict

**User testing (final phase):**
- [ ] Recruit 20–30 users matching target profile
- [ ] Run DIRECT experiment (blind comparison + creator survey)
- [ ] Run EDIT experiment (time + output comparison)
- [ ] Compile results in `docs/evaluation/results.md`
- [ ] Decision: improve Story Director quality, or advance to next phase?

**Ongoing:**
- [ ] Add new scenarios as new environments are tested
- [ ] Log all "why is it asking me this" moments from field testing
- [ ] Log all "I'd just use Camera" moments
- [ ] Version the evaluation dataset — treat it as a production asset
