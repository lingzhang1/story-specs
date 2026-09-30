# Story --- MVP Build Plan

**Status:** Post-UX-freeze implementation plan\
**Product goal:** Build the smallest real iPhone product that proves
Story can **plan, guide, manage, and create visual stories** for
ordinary users.

------------------------------------------------------------------------

# 1. Current State

Product definition and UX are sufficiently frozen to begin
implementation.

The MVP has three connected entry points:

``` text
HOME
├── Create Now
├── Start a Story
└── Use Existing Photos / Videos
```

The persistent Story experience includes:

``` text
Story Home
├── Moments
├── Shoot a Moment
├── Story Guide
├── Story Idea / Show Me
├── Create My Story
└── Complete / Reopen Story
```

The next work is not more broad UX invention.

The next work is:

``` text
CORE DOMAIN SCHEMA
      ↓
SYSTEM ARCHITECTURE
      ↓
WALKING SKELETON
      ↓
REAL AI ENGINES
      ↓
MEDIA CREATION
      ↓
FIELD EVALUATION
```

------------------------------------------------------------------------

# 2. Repository Structure

Use one monorepo.

``` text
story/
├── apps/
│   ├── ios/                         # Production iPhone app
│   └── api/                         # Story backend/API
│
├── packages/
│   ├── contracts/                   # Shared API/domain contracts
│   ├── story-engine/                # Persistent Story reasoning
│   ├── director-engine/             # Scene → concept → shot plan
│   ├── asset-engine/                # Photo/video understanding
│   ├── edit-engine/                 # Moments → EditPlan
│   ├── music-engine/                # Music recommendation/analysis
│   ├── media-engine/                # Deterministic rendering
│   ├── model-gateway/               # AI provider abstraction
│   └── observability/               # Logs/evals/traces
│
├── services/
│   └── worker/                      # Async rendering/heavy jobs when needed
│
├── prototypes/
│   └── figma/                       # Frozen Figma HTML/reference prototype
│       └── README.md
│
├── specs/
│   ├── story-production.md
│   ├── mvp-build-plan.md
│   ├── 02-core-story-schema.md
│   └── 03-system-architecture.md
│
├── evals/
│   ├── director/
│   ├── scene-understanding/
│   ├── asset-selection/
│   └── editing/
│
└── scripts/
```

The Figma HTML is reference code only.

Production code must not depend on `prototypes/figma`.

------------------------------------------------------------------------

# 3. Architecture Decisions to Freeze Before Coding Deeply

Before the walking skeleton expands, create `03-system-architecture.md`
and lock the following decisions:

-   iOS technology and application layering
-   backend language/framework
-   monorepo tooling
-   local persistence technology
-   API protocol
-   local vs cloud ownership
-   temporary media upload strategy
-   background job strategy
-   package dependency rules
-   model gateway interface
-   rendering approach
-   development/staging configuration
-   secrets management
-   observability
-   mock vs production engine interfaces

Do not design production microservices, Kubernetes, or large-scale
infrastructure.

For MVP, use one backend deployment with logical package boundaries.

The render worker can be split later if necessary.

------------------------------------------------------------------------

# 4. Core Domain Contract

`02-core-story-schema.md` is the canonical product-domain contract.

The implementation must preserve:

``` text
Story
├── CreativeIntent
├── MusicDirection
├── StoryBlueprint
├── StoryState
├── Moments[]
├── CaptureSessions[]
└── GeneratedOutputs[]
```

Important invariants:

``` text
Story != GeneratedOutput

Create My Story != Complete Story

Story-level guidance != Scene-specific direction
```

A Story can generate multiple outputs.

Original user media is local-first.

Do not let individual agents invent conflicting Story models.

------------------------------------------------------------------------

# 5. Logical Runtime Architecture

``` text
                    iPHONE
        ┌───────────────────────────┐
        │ SwiftUI                   │
        │ Camera / Photos           │
        │ Local Story persistence   │
        │ Local media references    │
        │ Playback                  │
        └─────────────┬─────────────┘
                      │ HTTPS
                      ▼
               ┌─────────────┐
               │  Story API  │
               └──────┬──────┘
                      │
       ┌──────────────┼─────────────────┐
       ▼              ▼                 ▼
 Story Engine   Director Engine     Edit Engine
       │              │                 │
       ├──────────────┼──────────┐      │
       ▼              ▼          ▼      ▼
 Asset Engine    Music Engine   Model Gateway
                                     │
                                  AI models

 EditPlan
    ↓
 Media Engine
    ↓
 GeneratedOutput
```

These are logical modules, not separate MVP services.

------------------------------------------------------------------------

# 6. Ownership Boundaries

## iOS owns

-   UI and navigation
-   camera capture
-   Photos integration
-   original local media
-   local Story persistence
-   Moment references
-   thumbnails/proxies
-   permissions
-   playback
-   lightweight client state

## Backend owns

-   orchestration
-   structured AI requests
-   model-provider access
-   Story/Director reasoning that requires server AI
-   temporary processing
-   async job coordination
-   edit planning
-   server-side rendering if selected

## Story Engine owns

-   StoryBlueprint
-   StoryState
-   Story Guide
-   Story Ideas
-   long-horizon Story reasoning

## Director Engine owns

-   current-scene reasoning
-   StoryConcept
-   StoryPlan
-   shot instructions
-   capture feasibility

## Asset Engine owns

-   semantic media understanding
-   quality/usability metadata
-   usable segments
-   future indexing foundation

## Music Engine owns

-   MusicDirection interpretation
-   candidate tracks
-   beat/section analysis

## Edit Engine owns

-   asset selection
-   narrative ordering
-   trim decisions
-   pacing
-   music-aware EditPlan

## Media Engine owns

-   deterministic execution of EditPlan
-   trim
-   concatenate
-   crop/reframe
-   speed
-   transitions
-   audio
-   export

## Model Gateway owns

-   provider abstraction
-   structured generation
-   vision calls
-   future embeddings/model routing

------------------------------------------------------------------------

# 7. Milestone A --- Walking Skeleton on a Real iPhone

This is the immediate implementation milestone.

The goal is not AI quality.

The goal is:

> **Every frozen product path works end-to-end on a real iPhone with
> mock intelligence and real local media/persistence.**

## A1. Project foundation

Build:

-   monorepo structure
-   iOS app
-   API skeleton
-   environment configuration
-   contracts package
-   mock engine interfaces
-   logging
-   basic error handling

## A2. Local identity and persistence

Implement:

-   anonymous local identity
-   Story creation
-   Story status
-   local Story persistence
-   Moments
-   CaptureSessions
-   GeneratedOutputs metadata

Test:

``` text
Create Story
→ add Moment
→ kill app
→ relaunch
→ Story and Moment still exist
```

## A3. Home

Implement frozen Home:

-   Create Now
-   Start a Story
-   Use Existing Photos / Videos
-   Continue
-   Your Stories

Active/completed organization must work.

## A4. Persistent Story

Implement:

-   Creative Direction
-   Story Home
-   Moments
-   add existing photos/videos
-   remove/preview Moment
-   Story Guide with mock content
-   Story Idea with mock content
-   Show Me
-   Shoot a Moment
-   Create My Story
-   Complete Story
-   Reopen Story

## A5. Real media plumbing

Implement real:

-   camera permission
-   photo-library permission
-   video capture
-   photo/video picker
-   local asset references
-   thumbnails
-   playback
-   local persistence

Do not upload originals by default.

## A6. Mock DIRECT

Use deterministic mock data for:

-   SceneUnderstanding
-   StoryConcept
-   StoryPlan
-   ValidationResult

Example:

``` text
Concept: Walk Into the Scale

Shot 1:
Keep the camera still.
Walk toward the ridge.
6 seconds.
```

## A7. Mock Create My Story

Implement:

-   preview available Moments
-   music selection UI
-   fake processing state
-   placeholder/mock GeneratedOutput

The Story status must remain unchanged.

## A8. Walking-skeleton acceptance test

A complete vertical slice must work:

``` text
Launch
→ Start a Story
→ define Creative Direction
→ Story Home
→ Shoot a Moment
→ mock idea
→ record real clip
→ save as Moment
→ Story Home
→ inspect Moments
→ Create My Story
→ mock result
→ return Home
→ kill app
→ relaunch
→ resume Story with state intact
```

Do not proceed to broad AI implementation until this works reliably.

------------------------------------------------------------------------

# 8. Milestone B --- Contracts and API Stabilization

Once the skeleton proves the object relationships, stabilize concrete
contracts.

Deliverables:

-   Swift models
-   backend models
-   API request/response schemas
-   package interfaces
-   serialization rules
-   error contracts
-   job-state contracts

Key objects:

-   Story
-   CreativeIntent
-   MusicDirection
-   StoryBlueprint
-   StoryState
-   Moment
-   Asset
-   AssetUnderstanding
-   CaptureSession
-   SceneUnderstanding
-   StoryConcept
-   StoryPlan
-   ShotPlan
-   ValidationResult
-   MusicAsset
-   MusicAnalysis
-   EditPlan
-   GeneratedOutput

Do not pass arbitrary prose between components where a structured
contract is appropriate.

------------------------------------------------------------------------

# 9. Milestone C --- Creative Intent

Replace mock intent handling with real AI.

Inputs:

``` text
template
+
user description
+
optional reference
+
context
+
desired duration
```

Output:

``` text
CreativeIntent
```

The output must be structured.

For reference media, MVP understanding can focus on:

-   mood
-   energy
-   pacing
-   average shot length
-   camera movement
-   subject/environment balance
-   general story grammar

Do not attempt perfect cinematic decomposition.

------------------------------------------------------------------------

# 10. Milestone D --- Scene Understanding

For scene-aware DIRECT:

``` text
3–5 second scan
      ↓
sample representative frames
      ↓
Vision model
      ↓
SceneUnderstanding
```

Extract only information needed for direction:

-   environment
-   subjects
-   lighting
-   motion opportunities
-   composition opportunities
-   constraints

Do not build full 3D scene reconstruction.

Latency matters. Avoid repeated expensive model calls during active
capture.

------------------------------------------------------------------------

# 11. Milestone E --- Director Engine

This is one of the highest-value MVP components.

Input:

``` text
CreativeIntent
+
optional Story context
+
SceneUnderstanding
+
Capture constraints
```

Output:

``` text
StoryConcept
+
StoryPlan
```

Optimize for:

``` text
creative quality
+
scene specificity
+
physical feasibility
+
low capture effort
```

MVP constraints:

``` text
max shots: 3
preferred shots: 1–2
capture time: roughly <= 30 seconds for short DIRECT
```

Recommend one strong default.

Alternatives are secondary.

------------------------------------------------------------------------

# 12. Milestone F --- Capture Guidance and Validation

Convert structured ShotPlans into simple sequential instructions.

Example:

``` text
Walk toward the signs.
↓
Ready?
↓
3... 2... 1...
↓
Go.
```

Validation should detect gross failure only:

-   clip too short
-   subject missing
-   catastrophic obstruction
-   severe shake
-   severe exposure problem
-   requested movement clearly missing

Output:

``` text
accept
```

or:

``` text
retake + simple reason
```

No aesthetic scores.

No "best take" ranking in MVP.

------------------------------------------------------------------------

# 13. Milestone G --- Persistent Story Intelligence

Replace mock Story Guide and Story Idea with real Story Engine behavior.

Input:

``` text
CreativeIntent
+
StoryBlueprint
+
StoryState
+
Moments / AssetUnderstanding
```

Output may include:

-   updated lightweight narrative coverage
-   Story Guide content
-   one useful Story Idea
-   recommendation to do nothing

The engine must understand that no intervention is often correct.

It should never require users to satisfy suggestions before creating.

------------------------------------------------------------------------

# 14. Milestone H --- Asset Understanding

Build media analysis shared by persistent Story and existing-media
creation.

For each asset, extract only useful MVP information:

-   duration
-   semantic description
-   scene
-   people/actions
-   basic quality
-   usable segment(s)
-   potential narrative role

For video, identify useful ranges where practical.

This package should be designed so it can later grow into large-library
indexing without building that system now.

------------------------------------------------------------------------

# 15. Milestone I --- Music Engine

Start with a small curated, licensing-safe catalog.

Implement:

-   MusicDirection
-   track metadata
-   mood matching
-   preview
-   small alternative set
-   BPM/beat analysis
-   basic section/energy analysis

Music is an input to edit planning, not merely an audio track added
after editing.

------------------------------------------------------------------------

# 16. Milestone J --- Edit Engine

Inputs:

``` text
CreativeIntent
+
StoryState
+
Moments / AssetUnderstanding
+
MusicAsset / MusicAnalysis
```

Output:

``` text
EditPlan
```

EditPlan should specify:

-   selected assets
-   usable ranges
-   ordering
-   output duration
-   speed
-   crop/reframe
-   transitions
-   music
-   cut timing

The AI chooses the edit.

It does not perform media operations itself.

------------------------------------------------------------------------

# 17. Milestone K --- Deterministic Media Engine

Implement only operations required by MVP:

-   trim
-   concatenate
-   crop/reframe
-   speed adjustment
-   simple transitions
-   music
-   basic beat alignment
-   export vertical MP4

No timeline editor.

No effect library.

No professional editing UI.

No proprietary generative video.

Acceptance criterion:

``` text
EditPlan
→ deterministic renderer
→ playable/exportable MP4
```

------------------------------------------------------------------------

# 18. Milestone L --- Existing Media Path

Once Asset + Edit + Music engines work, complete:

``` text
Use Existing Photos / Videos
→ select small set of assets
→ CreativeIntent
→ AssetUnderstanding
→ achievable Story direction
→ Music
→ EditPlan
→ GeneratedOutput
```

For MVP, keep selection intentionally small.

Do not solve thousands-of-assets ingestion yet.

------------------------------------------------------------------------

# 19. Evaluation Dataset

Start this before the Director Engine is considered finished.

Create a small scenario library such as:

-   busy city square at night
-   scenic overlook
-   beach sunset
-   mountain viewpoint
-   forest trail
-   café
-   restaurant
-   hotel room
-   downtown street
-   park
-   museum exterior
-   snow landscape
-   birthday dinner
-   family event

For each, test multiple intents:

-   cinematic
-   energetic
-   romantic
-   landscape-focused
-   playful
-   calm

Human evaluation questions:

-   Is the recommendation scene-specific?
-   Is it physically possible?
-   Would I actually shoot it?
-   Does it match the requested style?
-   Is it meaningfully different from repetitive defaults?
-   Is the capture effort justified?

Keep evaluation artifacts under `evals/`.

------------------------------------------------------------------------

# 20. Field Testing

Once real DIRECT works, test outside the office.

Test:

-   city street
-   scenic viewpoint
-   restaurant
-   park
-   beach
-   mountain/trail
-   indoor event

Record every moment where a tester says:

> "Why is it asking me to do this?"

or:

> "I would just use Camera."

Those are product failures even when the software technically works.

------------------------------------------------------------------------

# 21. Persistent Story Testing

Use one realistic multi-day scenario, such as TMB, to test the
persistent model.

Test:

``` text
create Story
→ inspect Story Guide
→ add existing Moment
→ shoot directed Moment
→ leave app
→ resume later
→ Story Idea
→ Show Me
→ capture
→ Create My Story
→ add more Moments
→ create another output
→ Complete Story
→ reopen Story
```

Verify that Create and Complete remain independent.

------------------------------------------------------------------------

# 22. Real User Test

Target users who:

-   travel or go out
-   care about good photos/videos
-   sometimes post Reels/short videos
-   are not professional videographers
-   do not enjoy serious editing

Test the three entry points separately and together.

For DIRECT, compare:

``` text
user's normal capture
vs
Story-directed capture
```

For persistent Story, evaluate:

-   whether Story Guide was useful before/during the experience
-   whether Story Ideas felt helpful or annoying
-   whether Moments made the experience easier to manage
-   whether users understood Create vs Complete
-   whether they created more than one output
-   whether they would use it on another trip/event

For existing media, compare time and output quality against the user's
normal workflow.

------------------------------------------------------------------------

# 23. Team / Agent Ownership

For a small human team using Kiro Crew, divide ownership by product
capability rather than "frontend vs backend."

## Product / AI owner

Own:

-   product decisions
-   CreativeIntent
-   Story Engine
-   Director quality
-   model selection/prompts
-   evaluation datasets
-   Figma/UX
-   user research

## Client / media owner

Own:

-   iOS
-   camera
-   Photos integration
-   local persistence
-   media ingestion
-   playback
-   deterministic video rendering

## Shared

Own:

-   core schema
-   system architecture
-   API contracts
-   field testing
-   metrics
-   major product decisions

Kiro can provide engineering bandwidth beneath these ownership
boundaries.

------------------------------------------------------------------------

# 24. Kiro Work Breakdown

Do not give Kiro a single prompt such as:

> "Build Story MVP."

Give it bounded tasks aligned with the architecture.

Suggested work areas:

``` text
01-project-foundation
02-core-story-schema
03-system-architecture
04-ios-shell-and-navigation
05-local-persistence
06-media-capture-and-picker
07-api-skeleton
08-creative-intent
09-scene-understanding
10-director-engine
11-capture-guidance
12-shot-validation
13-persistent-story
14-asset-understanding
15-music-engine
16-edit-engine
17-media-engine
18-existing-media-flow
19-evaluation
```

Each task should define:

-   objective
-   dependencies
-   owned files/packages
-   interfaces
-   acceptance criteria
-   tests
-   explicitly excluded work

Do not allow multiple agents to independently redefine shared domain
contracts.

------------------------------------------------------------------------

# 25. What to Fake First

For the walking skeleton, fake:

-   Story Guide intelligence
-   Story Idea intelligence
-   SceneUnderstanding
-   StoryConcept
-   StoryPlan
-   ValidationResult
-   EditPlan
-   GeneratedOutput
-   music recommendation

Use real:

-   iPhone navigation
-   camera
-   photo picker
-   local persistence
-   Moment storage
-   playback
-   Story lifecycle
-   permissions
-   real device behavior

This isolates architecture/UX failures before model quality becomes
involved.

------------------------------------------------------------------------

# 26. What Not to Build Yet

Do not add:

-   generative missing drone shots
-   thousands-of-photo indexing
-   automatic deduplication
-   deep personalization
-   AI avatars
-   AR overlays
-   full timeline editing
-   cloud photo library
-   mandatory accounts
-   social features
-   complex collaboration
-   third-party music catalogs
-   production microservices
-   sophisticated infrastructure

A feature enters MVP only if it helps test:

> **Can Story meaningfully plan, guide, manage, or create a user's
> visual story better than their normal camera + photo library + editing
> workflow?**

------------------------------------------------------------------------

# 27. Immediate Next Steps

The next work should happen in this order:

``` text
1. Finalize 02-core-story-schema.md
2. Write 03-system-architecture.md
3. Create monorepo/package skeleton
4. Move Figma HTML under prototypes/figma/
5. Add prototypes/figma/README.md declaring it reference-only
6. Build Milestone A walking skeleton
7. Run it on a real iPhone
8. Fix architecture/state/persistence issues
9. Stabilize concrete API/contracts
10. Begin CreativeIntent + SceneUnderstanding + Director Engine
```

The walking skeleton should start only after the core schema and
lightweight architecture decisions are clear enough that Kiro does not
invent them during implementation.

Do not wait for a perfect production architecture.

The goal is to define stable boundaries, then learn by building.

------------------------------------------------------------------------

# 28. First Major Acceptance Milestone

The first meaningful engineering milestone is:

> **A real iPhone can create a persistent Story, capture/import real
> media into Moments, leave and resume the Story, invoke a mock directed
> capture, preview current material, create a mock output,
> complete/reopen the Story, and retain everything after app restart.**

Once that works, the product architecture is real enough to begin
replacing mocks with intelligence.

The next major product milestone is:

> **Story gives a real scene-specific recommendation that makes the team
> say, "I genuinely would not have thought to shoot it this way."**

If that does not happen, improve the Director before expanding scope.

------------------------------------------------------------------------

# 29. MVP Success Definition

The MVP succeeds if evidence suggests that Story can provide value
across four connected jobs:

1.  **Plan** --- translate vague creative intent into useful Story
    direction.
2.  **Guide** --- help users capture better material when direction is
    worth the friction.
3.  **Manage** --- remember Moments and evolving Story state across an
    experience.
4.  **Create** --- turn available material into a music-aware finished
    visual story with little manual editing.

The product should feel like one AI Story Director across these jobs,
not four disconnected tools.
