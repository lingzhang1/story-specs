# 08 — Shot Validation

## Overview

After each captured clip, quickly check whether it satisfies the shot requirement. Output is binary: PASS or RETRY with a one-line plain-language reason. Shows while the user is standing there; must be fast and PASS-biased.

---

## Requirements

- Binary output only: PASS or RETRY — no scores, percentages, or quality ratings shown to user
- Check gross failures only: clip long enough, subject visible, no catastrophic obstruction, camera reasonably stable, expected motion roughly occurred
- RETRY reason is one plain-language sentence: "Someone walked in front of you halfway through." Not "subject occlusion at t=2.3s."
- Latency: < 3 seconds from clip capture completing
- **PASS-biased:** when uncertain, return PASS — a false PASS (minor clip issues) is better than a false RETRY (user retakes a perfectly fine clip). Edit Planner handles minor quality issues.
- Retake is always optional; user can proceed regardless of RETRY verdict

---

## Design

**Pipeline:**
```
Captured clip + shot requirement
        ↓
Duration check (AVFoundation): clip ≥ shot.duration × 0.7?
  +
Motion energy heuristic: catastrophically unstable?
  +
3 frame samples (start, middle, end)
        ↓
VLM call: "Did this satisfy the requirement? Err strongly toward PASS."
        ↓
{ result: "pass" | "retry", reason: "string | null" }
```

**VLM prompt:** "You are reviewing a captured video clip. The shot required: [purpose], [subject_action], [camera_action]. Review the frames. Only return RETRY for obvious failures — subject never visible, completely wrong scene, camera completely unusable. If any doubt, return PASS."

**UX:**
```
"Got it." (immediate feedback)
  ↓
Brief spinner (< 3 sec)
  ↓
✓ "Nice."                    or    ↺ "Let's try that again."
                                       "Someone walked in front of you."
  ↓                                      ↓
Next shot                      [Try again]  [It's fine, continue]
```

---

## Tasks

**Phase 1 (fake):**
- [ ] Always return PASS after 1-second fake delay
- [ ] Show "Nice." and proceed

**Phase 5 (real AI):**
- [ ] Implement duration check
- [ ] Implement motion energy heuristic
- [ ] Implement 3-frame sampler
- [ ] Design and test VLM prompt → `{ result, reason }`
- [ ] Tune PASS bias: measure false retry rate on 20 test clips
- [ ] Measure latency; must be < 3s
- [ ] Build RETRY UX with plain-language reason and optional retake
