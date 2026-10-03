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

| Feature / surface | Figma location | Capability | C3.5 UI treatment | Why | Dependency | Target milestone |
|---|---|---|---|---|---|---|
| Creative Intent (free text → resolved intent) | Screen1Intent / ScreenStoryDirection | IMPLEMENTED (C1) | WIRED | — | — | done |
| Scene scan → SceneUnderstanding | Screen3Scene / ScreenTMBShootScene | IMPLEMENTED (C2) | WIRED | — | — | done |
| Story Director idea/recommendation | Screen5Idea / ScreenTMBShootIdea | IMPLEMENTED (C3) | WIRED | — | — | done |
| Director thinking state | Screen4Thinking / ScreenTMBShootThinking | IMPLEMENTED | WIRED (real 12–16s) | — | — | done |
| capture_shot → shot guidance → camera | ShotScreen + CaptureScreen | IMPLEMENTED | WIRED | — | — | done |
| rescan_scene (decline scene) | (REAL_SYSTEM_EXTENSION — not in Figma) | IMPLEMENTED (C3) | WIRED | real Director state | — | done |
| create_now (enough material) decision | (REAL_SYSTEM_EXTENSION) | PARTIAL (decision only) | WIRED decision + DISABLED downstream generate | real generation absent | Edit Planner + Media | later Create-Story milestone |
| Moment capture → persist | ShotScreen→Story | IMPLEMENTED (A) | WIRED | — | — | done |
| **"Try another idea"** (concept variation) | Screen5Idea / ScreenTMBShootIdea | NOT_IMPLEMENTED | DISABLED | needs a Director variation re-call path | Director variation call | C4+ |
| **Skip scan** (intent-only idea) | Screen3Scene / ShotScreen skip | NOT_IMPLEMENTED | DISABLED | Director has no intent-only (no-scene) path | Director intent-only path | C4+ |
| **Shot Validation** (success/retry) | Screen8Validation | NOT_IMPLEMENTED | DISABLED | no real validation | Shot Validation | **C4** |
| **Validation-driven retry** | Screen8Validation retry | NOT_IMPLEMENTED | DISABLED | depends on validation | Validation + Director loop | **C4** |
| **Real Beat-progress feedback** | ScreenTMB Story Guide states | NOT_IMPLEMENTED | DISABLED (static advisory copy, no live ✓) | needs captured-video→BeatProgress | Director loop + AssetUnderstanding | C4+ |
| **Existing media — analyze path** (pre-shoot) | ScreenMediaPicker(analyze) + PreMediaResult A/B/C | NOT_IMPLEMENTED | DISABLED entry | no AssetUnderstanding | AssetUnderstanding | later |
| **Existing media — add-to-Moments path** | ScreenMediaPicker(add) from Moments/Preview | PARTIAL (can import file as Moment) | WIRED if trivial else DISABLED | import plumbing exists for captures; picker not built | PHPicker integration | C3.5 if cheap, else later |
| **Music direction (feeling)** | MusicDirectionRow/Sheet | NOT_IMPLEMENTED | DISABLED | no music model | Music Engine | later Create-Story milestone |
| **Music track selection / re-edit** | ScreenTMBMusic | NOT_IMPLEMENTED | DISABLED | no music model/engine | Music Engine | later |
| **Create My Story → real generation** | Screen10Creating → Screen11Result | NOT_IMPLEMENTED (mock output only, A) | DISABLED real-generate; mock DEBUG_ONLY | no Edit Planner/Media Engine | Edit Planner + Media | later |
| **Generated Output result (play/save/share)** | Screen11Result | NOT_IMPLEMENTED (mock) | EMPTY_STATE + DISABLED actions | no real output | Edit Planner + Media | later |
| **Create a new version / re-edit** | Screen11Result | NOT_IMPLEMENTED | DISABLED | depends on generation | Edit Planner + Media | later |
| **Complete Story (lifecycle)** | ScreenTMB menu → Complete/Reopen | NOT_IMPLEMENTED (status enum exists; no UI) | WIRE if trivial (status flip + sort) else DISABLED | status model exists (active/completed); menu UI absent | — (local state only) | C3.5 (cheap) |
| **Story Guide / arc ("what I'll look for")** | ScreenStoryBlueprint / ScreenTMB Guide | NOT_IMPLEMENTED (no StoryBlueprint model) | DISABLED (static arc from Figma copy) | no persistent StoryBlueprint intelligence | StoryBlueprint/Guide milestone | later |
| **Story Idea passive suggestion** | ScreenTMB "✦ Story Idea" | NOT_IMPLEMENTED | DISABLED | needs persistent-Story orchestration | Story Guide milestone | later |
| **Reference media (vibe from a clip)** | Screen1Intent / ScreenStoryDirection reference | NOT_IMPLEMENTED | DISABLED | reference transport not built (C1 is text-only) | reference-ingest | later |

## Rule
When a milestone implements any NOT_IMPLEMENTED/PARTIAL row, it must: (1) wire the
corresponding Figma surface, (2) flip the row here to IMPLEMENTED/WIRED, (3) note
the enabling milestone. This file is the single source of "what's intentionally
disabled and why" so no frozen capability is silently dropped.
