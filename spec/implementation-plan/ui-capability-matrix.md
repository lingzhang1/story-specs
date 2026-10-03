# UI Capability Matrix — frozen Figma surfaces vs implemented capability

**Purpose:** every capability the frozen Figma UX (`prototypes/story-figma/src/App.tsx`)
presents, with its current backend status, why it is disabled if it is, the
dependency, and the milestone expected to enable it. A future milestone that
implements a capability MUST, in its Definition of Done, enable the corresponding
Figma UI here and flip the row.

**Frozen Figma artifact:** `prototypes/story-figma/src/App.tsx` (4730 lines, single
Figma Make export, git `ba88526`). No competing version.

**Legend — Capability:** IMPLEMENTED · PARTIAL · NOT_IMPLEMENTED.
**UI treatment at C3.5:** WIRED (real) · DISABLED (visible, non-interactive) ·
DEBUG_ONLY · EMPTY_STATE.
**Figma parity:** MATCHED · PARTIAL · MISSING · REAL_SYSTEM_EXTENSION. C3.5
Definition of Done requires **zero unexplained MISSING rows** — every MISSING must
carry a dependency + target milestone.

| Feature / surface | Figma location | Capability | C3.5 UI treatment | Figma parity | Why | Dependency | Target milestone |
|---|---|---|---|---|---|---|---|
| Creative Intent (free text → resolved intent) | Screen1Intent / ScreenStoryDirection | IMPLEMENTED (C1) | WIRED | MATCHED | — | — | done |
| Scene scan → SceneUnderstanding | Screen3Scene / ScreenTMBShootScene | IMPLEMENTED (C2) | WIRED | MATCHED | — | — | done |
| Story Director idea/recommendation | Screen5Idea / ScreenTMBShootIdea | IMPLEMENTED (C3) | WIRED | MATCHED | — | — | done |
| Director thinking state | Screen4Thinking / ScreenTMBShootThinking | IMPLEMENTED | WIRED (real 12–16s) | REAL_SYSTEM_EXTENSION | staged copy for real latency | — | done |
| capture_shot → shot guidance → camera | ShotScreen + CaptureScreen | IMPLEMENTED | WIRED | MATCHED | — | — | done |
| rescan_scene (decline scene) | (not in Figma) | IMPLEMENTED (C3) | WIRED | REAL_SYSTEM_EXTENSION | real Director state | — | done |
| create_now (enough material) decision | (not in Figma) | PARTIAL (decision only) | WIRED decision + DISABLED downstream generate | REAL_SYSTEM_EXTENSION | real generation absent | Edit Planner + Media | later Create-Story milestone |
| Moment capture → persist | ShotScreen→Story | IMPLEMENTED (A) | WIRED | MATCHED | — | — | done |
| **"Try another idea"** (concept variation) | Screen5Idea / ScreenTMBShootIdea | NOT_IMPLEMENTED | DISABLED | PARTIAL | needs a Director variation re-call path; NOT a validation feature | Director enhancement / TBD | TBD (not adopted by a milestone yet) |
| **Skip scan** (intent-only idea) | Screen3Scene / ShotScreen skip | NOT_IMPLEMENTED | DISABLED | PARTIAL | Director has no intent-only (no-scene) path; NOT a validation feature | Director enhancement / TBD | TBD (not adopted by a milestone yet) |
| **Shot Validation** (success/retry) | Screen8Validation | NOT_IMPLEMENTED | DISABLED | MISSING | no real validation | Shot Validation | **C4** |
| **Validation-driven retry** | Screen8Validation retry | NOT_IMPLEMENTED | DISABLED | MISSING | depends on validation | Validation + Director loop | **C4** |
| **Real Beat-progress feedback** | ScreenTMB Story Guide states | NOT_IMPLEMENTED | DISABLED (neutral unavailable, no live ✓) | PARTIAL | needs captured-video→BeatProgress | Director loop + AssetUnderstanding | later |
| **Existing media — analyze path** (pre-shoot) | ScreenMediaPicker(analyze) + PreMediaResult A/B/C | NOT_IMPLEMENTED | DISABLED entry | MISSING | no AssetUnderstanding | AssetUnderstanding | later |
| **Existing media — add-to-Moments path** | ScreenMediaPicker(add) from Moments/Preview | PARTIAL (import plumbing exists) | **WIRED (C3.5)** | PARTIAL | local/mechanical: PHPicker → append Moment, no analysis | PHPicker | **C3.5** |
| **Music direction (feeling)** | MusicDirectionRow/Sheet | NOT_IMPLEMENTED | DISABLED | MISSING | no music model | Music Engine | later Create-Story milestone |
| **Music track selection / re-edit** | ScreenTMBMusic | NOT_IMPLEMENTED | DISABLED | MISSING | no music model/engine | Music Engine | later |
| **Create My Story → real generation** | Screen10Creating → Screen11Result | NOT_IMPLEMENTED (mock output only, A) | DISABLED real-generate; mock DEBUG_ONLY | PARTIAL | no Edit Planner/Media Engine | Edit Planner + Media | later |
| **Generated Output result (play/save/share)** | Screen11Result | NOT_IMPLEMENTED (mock) | EMPTY_STATE + DISABLED actions | PARTIAL | no real output | Edit Planner + Media | later |
| **Create a new version / re-edit** | Screen11Result | NOT_IMPLEMENTED | DISABLED | MISSING | depends on generation | Edit Planner + Media | later |
| **Complete Story (lifecycle) + Reopen** | ScreenTMB menu → Complete/Reopen | PARTIAL (status enum exists) | **WIRED (C3.5)** | PARTIAL | local/mechanical: flip status active↔completed + sort completed below active | — (local state only) | **C3.5** |
| **Story Guide / arc ("what I'll look for")** | ScreenStoryBlueprint / ScreenTMB Guide | NOT_IMPLEMENTED (no StoryBlueprint model) | DISABLED (neutral unavailable state — NOT TMB demo copy) | PARTIAL | no persistent StoryBlueprint intelligence; Figma TMB chapters/categories are demonstration content, not production-generated | StoryBlueprint/Guide milestone | later |
| **Story Idea passive suggestion** | ScreenTMB "✦ Story Idea" | NOT_IMPLEMENTED | DISABLED (neutral unavailable state — NOT TMB demo copy) | PARTIAL | needs persistent-Story orchestration; TMB "A moment together" is demo content | Story Guide milestone | later |
| **Reference media (vibe from a clip)** | Screen1Intent / ScreenStoryDirection reference | NOT_IMPLEMENTED | DISABLED | MISSING | reference transport not built (C1 is text-only) | reference-ingest | later |

## Note on Story Guide / Story Idea (review correction)
The frozen Figma Story Guide chapters (Departure / On the Trail / Scale / …) and the
"✦ Story Idea — A moment together" card are **TMB demonstration content**, not
production-generated personalized Story content. C3.5 preserves their Figma
structure and visual hierarchy but renders them in a **neutral, clearly
unavailable** state (e.g. a muted "Story Guide — coming soon" treatment), NEVER
pasting the TMB copy over an arbitrary Story as if it were real guidance.

## Rule
When a milestone implements any NOT_IMPLEMENTED/PARTIAL row, it must: (1) wire the
corresponding Figma surface, (2) flip the row here to IMPLEMENTED/WIRED, (3) note
the enabling milestone. This file is the single source of "what's intentionally
disabled and why" so no frozen capability is silently dropped.
