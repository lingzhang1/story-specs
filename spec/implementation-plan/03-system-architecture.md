# Story — System Architecture

**Status:** MVP decisions frozen  
**Complements:** `mvp-build-plan.md`, `02-core-story-schema.md`  
**Purpose:** Lock the deployment, language, and infrastructure decisions before deep implementation begins. This document answers every decision listed in `mvp-build-plan.md § 3`.

---

## 1. Summary of Decisions

| Decision | Choice |
|---|---|
| iOS technology | Swift / SwiftUI |
| Backend language | Python 3.12 |
| Infrastructure-as-code | AWS CDK (TypeScript) |
| API layer | AWS App Runner (always-on, supports streaming) |
| Async / render jobs | Amazon SQS + AWS Lambda (Fargate for Media Engine if needed) |
| Media storage | Amazon S3 |
| Server-side persistence | Amazon DynamoDB |
| Secrets | AWS Secrets Manager |
| Observability | Amazon CloudWatch + structured JSON logging |
| Model access | AWS Bedrock (primary) + direct provider APIs via Model Gateway |
| Monorepo tooling | Turborepo (JS/TS packages) + Hatch / uv (Python packages) |
| API protocol | HTTPS / REST + JSON; streaming responses via SSE |
| Local vs cloud ownership | iOS owns originals; cloud owns reasoning, processing, outputs |
| Development / staging | CDK environments (dev / prod stacks, `.env` per environment) |
| Mock vs production engines | Interface-stable mock engines for Milestone A; real engines swapped in per milestone |

---

## 2. Monorepo Structure

```text
story/
├── apps/
│   ├── ios/                         # Swift/SwiftUI iPhone app (Xcode project)
│   └── api/                         # Python backend (FastAPI, entry point for App Runner)
│
├── packages/
│   ├── contracts/                   # Shared API/domain contracts (TypeScript types + JSON Schema)
│   ├── story-engine/                # Python: persistent Story reasoning
│   ├── director-engine/             # Python: scene → concept → shot plan
│   ├── asset-engine/                # Python: photo/video understanding
│   ├── edit-engine/                 # Python: Moments → EditPlan
│   ├── music-engine/                # Python: music recommendation / analysis
│   ├── media-engine/                # Python: deterministic rendering (FFmpeg)
│   ├── model-gateway/               # Python: AI provider abstraction
│   └── observability/               # Python: structured logging, traces
│
├── services/
│   └── worker/                      # Python: SQS consumer for async render jobs
│
├── infra/                           # AWS CDK app (TypeScript)
│   ├── bin/
│   │   └── story.ts                 # CDK entrypoint
│   └── lib/
│       ├── api-stack.ts             # App Runner service, DynamoDB, Secrets
│       ├── storage-stack.ts         # S3 buckets, bucket policies
│       ├── worker-stack.ts          # SQS queue + Lambda / Fargate worker
│       └── observability-stack.ts   # CloudWatch dashboards, alarms
│
├── prototypes/
│   └── story-figma/                 # Frozen Figma HTML prototype (reference only)
│
├── specs/                           # All spec documents
├── evals/                           # Model evaluation harnesses
└── scripts/                         # Dev / CI helper scripts
```

Production code must not import from `prototypes/`.

---

## 3. iOS Application

- **Language:** Swift 5.10+
- **UI framework:** SwiftUI
- **Local persistence:** SwiftData (preferred) or Core Data — chosen at Milestone A
- **Media references:** local file URLs only; originals never leave the device by default
- **Uploads:** iOS generates a presigned S3 URL via the API; uploads directly to S3 over HTTPS; sends the S3 object key back to the API for processing
- **Networking:** `URLSession` + async/await; SSE for streaming AI responses
- **Camera / Photos:** `AVFoundation` (capture), `PhotosUI` (picker)

### iOS owns

- UI and navigation
- Camera capture and Photos integration
- Original local media (never uploaded automatically)
- Local Story persistence (all Story/Moment/CaptureSession state)
- Thumbnails and proxies (generated on device)
- Playback of local and downloaded GeneratedOutputs
- Permissions (camera, photo library, microphone)
- Lightweight client state (current screen, in-progress capture)

---

## 4. Backend — Story API

- **Language:** Python 3.12
- **Framework:** FastAPI
- **Deployment:** AWS App Runner
  - Always-on (no cold starts); supports SSE streaming responses
  - Auto-scales based on request concurrency
  - Pulls image from Amazon ECR on deploy
- **API protocol:** HTTPS REST + JSON
- **Streaming:** Server-Sent Events (SSE) for Director Engine responses and long AI calls
- **Auth:** Anonymous device identity (`LocalIdentity.deviceId`) for MVP; no accounts required

### Why App Runner over Lambda for the API

Lambda + API Gateway can stream, but it adds configuration complexity (function URLs, response streaming flag, payload size limits). App Runner is a single container, easier to reason about locally and in staging, and has no 29-second API Gateway timeout. The API can be placed behind API Gateway later if needed.

### Backend owns

- Orchestration across engines
- Structured AI requests (via Model Gateway)
- Model-provider access and retries
- Story/Director reasoning requiring server AI
- Temporary media processing coordination
- Async job dispatch (SQS)
- Edit planning
- Server-side rendering coordination

---

## 5. Persistence — Amazon DynamoDB

Single-table design with entity types for `Story`, `Moment`, `CaptureSession`, `GeneratedOutput`.

- **Partition key:** `entityId` (e.g., `STORY#<storyId>`)
- **Sort key:** `sk` (e.g., `META`, `MOMENT#<momentId>`)
- **Access patterns:** get Story by id, list Moments by Story, list GeneratedOutputs by Story
- **TTL:** on temporary processing records (SceneUnderstanding, transient state)

DynamoDB suits the document-shaped Story model and scales to zero between uses (cost-efficient for MVP).

> **If relational access patterns emerge** (cross-Story queries, analytics), add Amazon Aurora Serverless v2 then. Do not add it preemptively.

---

## 6. Media Storage — Amazon S3

Two buckets:

| Bucket | Purpose | Lifecycle |
|---|---|---|
| `story-media-uploads` | Temporary: iOS upload landing zone, processing inputs | 7-day TTL on all objects |
| `story-media-outputs` | Persistent: GeneratedOutput MP4s, thumbnails | Retained; user-deletable |

- iOS uploads via presigned URL (5-minute expiry); no AWS credentials on device
- Media Engine writes GeneratedOutputs to `story-media-outputs`
- Presigned download URLs for GeneratedOutput delivery to iOS
- CORS configured for direct iOS upload/download

---

## 7. Async Jobs — SQS + Lambda (Worker)

The Media Engine (FFmpeg rendering) is CPU/memory-heavy and unsuitable for the API request path.

```text
API  →  SQS queue (story-render-jobs)  →  Worker
                                              │
                                     Lambda (< 5 min jobs)
                                              OR
                                     Fargate task (> 5 min / high memory)
```

- Start with **Lambda** (up to 10GB memory, 15-min timeout, sufficient for short clips)
- Switch the worker to **Fargate** if rendering times or memory regularly exceed Lambda limits
- Worker polls SQS, processes the `EditPlan`, writes MP4 to S3, updates DynamoDB, notifies iOS via a polling endpoint or push notification

Job status is stored in DynamoDB (`GeneratedOutput.status`). iOS polls `/outputs/{outputId}/status` until `ready`.

---

## 8. AI Models — Model Gateway

All AI calls go through the `model-gateway` package. No engine calls a provider directly.

```text
Engine  →  model-gateway  →  AWS Bedrock  →  Claude / Titan / etc.
                          →  OpenAI API   →  GPT-4o / o3
                          →  Anthropic API (direct, if not via Bedrock)
```

- **Primary:** AWS Bedrock (Claude 3.x for reasoning, multimodal for vision)
- **Fallback / alternative:** direct Anthropic or OpenAI API keys via Secrets Manager
- Model routing is internal to `model-gateway`; engines declare capability requirements, not model names
- Structured generation (JSON mode / tool use) is the default for all engine calls
- Vision calls (SceneUnderstanding, AssetUnderstanding) pass S3 object URLs or base64 blobs

---

## 9. Secrets — AWS Secrets Manager

| Secret name | Contents |
|---|---|
| `story/api-keys/bedrock` | Bedrock region / role ARN |
| `story/api-keys/openai` | OpenAI API key (fallback) |
| `story/api-keys/anthropic` | Anthropic API key (fallback) |
| `story/s3/bucket-names` | Upload / output bucket names |
| `story/dynamodb/table-name` | DynamoDB table name |

Secrets are injected into App Runner and Lambda via IAM role + Secrets Manager integration. No secrets in environment variables or source code.

---

## 10. Infrastructure-as-Code — AWS CDK (TypeScript)

All AWS resources are defined in `infra/` as CDK stacks.

### Stacks

| Stack | Resources |
|---|---|
| `ApiStack` | App Runner service, ECR repository, DynamoDB table, IAM roles |
| `StorageStack` | S3 upload and output buckets, bucket policies, CORS |
| `WorkerStack` | SQS queue, Lambda worker function (or Fargate task definition) |
| `ObservabilityStack` | CloudWatch log groups, dashboards, alarms |

### Environments

```typescript
// infra/bin/story.ts
new ApiStack(app, 'StoryApi-Dev',  { env: { account: '...', region: 'us-east-1' }, stage: 'dev'  });
new ApiStack(app, 'StoryApi-Prod', { env: { account: '...', region: 'us-east-1' }, stage: 'prod' });
```

Dev and prod are separate CDK deployments. No shared state between environments.

### Deploy commands

```bash
cd infra
npm install
npx cdk bootstrap   # once per account/region
npx cdk deploy --all --context stage=dev
npx cdk diff        # preview changes before deploy
```

---

## 11. Observability

- **Structured logging:** all Python packages log JSON via the `observability` package; log level from env (`LOG_LEVEL`)
- **CloudWatch Logs:** App Runner and Lambda forward logs automatically
- **CloudWatch Metrics:** custom metrics for AI call latency, render job duration, error rates
- **Alarms:** error rate > 1% over 5 minutes → SNS notification
- **Traces:** AWS X-Ray on App Runner and Lambda (opt-in, enabled in staging/prod)
- **Evals:** separate from observability; eval harnesses live in `evals/` and run offline against logged inputs/outputs

---

## 12. Local Development

```text
iOS app  →  localhost:8000  →  FastAPI dev server  →  mock engines
```

- `apps/api` runs locally with `uvicorn apps.api.main:app --reload`
- Mock engines return deterministic data (Milestone A)
- No AWS services required for Milestone A local dev
- DynamoDB Local (Docker) optional for local persistence testing
- S3 can be mocked with LocalStack or skipped (use local file paths) during skeleton phase

Environment config:

```text
apps/api/.env.dev      # local dev
apps/api/.env.staging  # staging (points to dev AWS stack)
apps/api/.env.prod     # production (injected by App Runner from Secrets Manager)
```

---

## 13. Package Dependency Rules

```text
ios app          → contracts (Swift models only)
api              → story-engine, director-engine, edit-engine, music-engine, asset-engine, model-gateway, observability
story-engine     → model-gateway, observability
director-engine  → model-gateway, observability
asset-engine     → model-gateway, observability
edit-engine      → model-gateway, observability
music-engine     → model-gateway, observability
media-engine     → observability (no AI calls; executes EditPlan deterministically)
model-gateway    → observability
worker           → edit-engine, media-engine, observability
contracts        → (no internal dependencies; pure types)
```

No engine imports another engine. All cross-engine coordination goes through `api` or `worker`.

---

## 14. Rendering Approach

- **Primary:** server-side via Media Engine (FFmpeg) running in the worker
- **On-device rendering:** not in MVP scope; may be added later (AVComposition)
- EditPlan is a deterministic specification; the same EditPlan produces the same output regardless of where it runs
- Media Engine has no AI calls; it only executes what Edit Engine planned

---

## 15. Mock vs Production Engine Interfaces

Each engine package exports a stable Python protocol/interface. Milestone A uses mock implementations behind those interfaces. Real implementations replace mocks per milestone without changing the API contract.

```python
# packages/director-engine/director_engine/interface.py
class DirectorEngine(Protocol):
    async def analyze_scene(self, input: SceneInput) -> SceneUnderstanding: ...
    async def generate_concept(self, input: ConceptInput) -> StoryConcept: ...
    async def generate_shot_plan(self, input: ShotPlanInput) -> StoryPlan: ...

# Milestone A
class MockDirectorEngine:
    async def analyze_scene(self, input: SceneInput) -> SceneUnderstanding:
        return SceneUnderstanding(description="Mock scene", ...)

# Milestone C (real)
class BedrockDirectorEngine:
    async def analyze_scene(self, input: SceneInput) -> SceneUnderstanding:
        ...  # real Bedrock call
```

The `api` package is always wired to an interface, never a concrete implementation directly.

---

## 16. Explicitly Out of Scope for MVP

- Kubernetes / EKS
- GraphQL
- WebSockets (SSE is sufficient)
- Multi-region deployment
- CDN (CloudFront) — add when GeneratedOutput delivery latency matters
- User accounts or authentication (anonymous identity only)
- Push notifications (polling is sufficient for MVP output status)
- On-device AI inference
- Android
