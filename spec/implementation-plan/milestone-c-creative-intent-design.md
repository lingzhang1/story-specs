# Milestone C — Creative Intent: Implementation Design

**Status:** Approved for implementation (design only — no real AI built yet).
**Date:** 2026-10-01
**Component:** Creative Intent Resolver
**Owner:** Story Engine (backend)
**Output contract:** `CreativeIntent` (`contracts/schemas/creative-intent.schema.json`)
**Supersedes for implementation:** the exploratory draft `04-creative-intent.md`
(kept as design rationale). Where the two differ, THIS document and the frozen
schema win.

This is the first real-AI component of Milestone C. It replaces the Milestone-A
mock `derive_creative_intent` with a server-side, model-backed, schema-validated
resolver — without changing the frozen four-layer discipline.

---

## 1. Scope & decisions (what this design locks)

Adopted from the `04-creative-intent.md` semantics, with four amendments the
team approved:

1. **`pace` is a first-class field** — categorical `slow | moderate | fast`.
   Director benefits from an explicit pacing signal; inferring it from tone is
   lossy. Not exact edit timing.
2. **`creative_constraints` replaces `avoid`** — holds positive AND negative
   preferences ("authentic, not staged"; "epic but not cheesy").
3. **One canonical schema.** Field names are exactly the frozen
   `CreativeIntentContract`; this doc does not invent a competing field list.
4. **Typed Model Gateway contract** with bounded structured-output repair/retry
   (defined in §6) — not an untyped `dict -> dict`.

Two architecture decisions (approved):

- **Resolution is server-side, always.** The resolver needs the Model Gateway
  + provider credentials, which may only live on the backend (03 §15, §40 forbid
  iOS from touching an AI provider). The device keeps its local mock ONLY as an
  offline/degraded fallback (§9), never as a second real implementation.
- **Hybrid sync/async, split by input type** (§5): text/template resolution is
  synchronous; reference-media resolution is an async job (presigned upload +
  poll), because multimodal analysis can exceed the App Runner budget.

No change to: the four model layers, local-first media, Create ≠ Complete,
`active|completed`, or any existing Milestone-A runtime path.

---

## 2. The frozen CreativeIntent contract (authoritative field list)

Source of truth: `contracts/schemas/ai_contracts.py :: CreativeIntentContract`
→ `contracts/schemas/creative-intent.schema.json` → Swift
`CreativeIntentContractDTO`. Fields:

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | opaque |
| `source_inputs[]` | `CreativeIntentSource` | `text` / `template` / `reference_asset` |
| `summary` | string (required) | one-line normalized intent |
| `subject?` | string | what the Story is about (semantic, not detected objects) |
| `desired_feeling[]` | string[] | small-vocab tone (energetic/calm/cinematic/…) |
| `style_tags[]` | string[] | visual-character tags |
| `pace?` | enum `slow\|moderate\|fast` | **new** |
| `emphasis[]` | string[] | person/environment/scale/journey/… |
| `creative_constraints[]` | string[] | **widened** — positive+negative preferences |
| `target_duration_seconds?` | number (>0) | explicit / template / product default |
| `target_format?` | enum `vertical\|square\|landscape` | |
| `raw_user_description?` | string | preserved user language |
| `created_at` | datetime | ISO-8601 UTC |

`desired_feeling` + `style_tags` together are the design's "tone" (split so
Director consumes feeling and Music/visual style consume their slice
independently). All JSON is **snake_case**; `extra="forbid"` — unknown keys are
rejected, so AI output cannot silently add fields.

**Must NOT contain** (enforced by the schema's closed shape, per 04 §8): scene
facts, shot/edit instructions, or `modelId/provider/promptVersion/requestId`
(those are operational metadata, §8).

---

## 3. Component placement

```
iOS (Creative Direction UI)
   │  HTTPS, snake_case JSON
   ▼
Story API  (apps/api)  POST /v1/creative-intent
   │
   ▼
Story Engine → Creative Intent Resolver        (packages/story_engine)
   │
   ▼
Model Gateway (packages/model_gateway)  ──► configured provider (Bedrock/external)
```

- The resolver lives in `story_engine` (it is long-horizon Story reasoning's
  entry). It depends ONLY on `contracts` + `model_gateway` (allowed chain, 03 §40).
- The resolver NEVER imports a provider SDK; all model access is through the
  gateway. Model names are configuration (03 §15).
- iOS calls the endpoint; it does not resolve real intent locally.

---

## 4. Resolution pipeline

```
CreativeDirectionInput { text?, template_id?, reference_asset_id?, context }
        │
        ▼
(1) Input normalization        — deterministic
        │   template expansion (lookup → structured priors)
        │   explicit settings pass-through (duration, format, selected vibe)
        ▼
(2) Reference analysis         — model, ONLY if reference_asset_id present (ASYNC)
        │   → ReferenceCharacteristics (energy, pace, shot-length, movement, …)
        ▼
(3) Intent resolver            — model, structured output
        │   combine sources by precedence (§5); preserve nuance
        ▼
(4) Schema validation          — Pydantic + semantic checks; bounded repair (§6)
        ▼
    CreativeIntent  (canonical, schema-valid)
```

Steps (1) and (4) are deterministic. Steps (2) and (3) are model-based (§7 of
the exploratory draft). Reference analysis is logically separate from intent
resolution even if one model backs both initially.

**Deterministic-first rule:** never call a model to transform an already-explicit
value (e.g. `duration=10` → `target_duration_seconds=10`). Product defaults
(e.g. Create Now short-video duration) are supplied to the resolver as context,
not invented by the model (§13 of the draft).

---

## 5. Transport: hybrid sync/async

### 5.1 Text / template → synchronous
`POST /v1/creative-intent`
```jsonc
// request
{
  "sources": [
    {"type": "text", "value": "epic but not cheesy hike, make us feel tiny"},
    {"type": "template", "template_id": "cinematic-journey"}   // optional
  ],
  "context": {
    "mode": "create_now" | "start_story",
    "story_title": "Tour du Mont Blanc",        // optional
    "target_duration_seconds": 10,              // optional explicit
    "target_format": "vertical"                 // optional explicit
  }
}
// 200 response: a canonical CreativeIntent (creative-intent.schema.json)
```
Single fast structured-output call. Target latency ~1–3s (§8), well inside the
120s App Runner budget. No queue — the infra doc forbids introducing SQS before
measured need.

### 5.2 Reference media → asynchronous (job + poll)
Multimodal analysis can be slow and needs frames uploaded. Flow:
```
iOS → POST /v1/uploads               → presigned target + opaque key (04-infra §16)
iOS → upload proxy/frames directly to S3 (bytes never cross the app server)
iOS → POST /v1/creative-intent  with  source {type:"reference_asset", asset_id}
        → 202 Accepted  { job_id }
backend: Job(queued→running) → reference analysis → intent resolve → validate
iOS → GET /v1/creative-intent/jobs/{job_id}   (poll; SSE optional later)
        → { state: queued|running|succeeded|failed, result?: CreativeIntent }
```
Job states reuse the frozen vocabulary `queued|running|succeeded|failed|cancelled`
(03 §31). Media constraints: client generates task-appropriate proxies/frames
before upload; API rejects unsupported type/size (04 §32, §44). S3 objects are
private, opaque-keyed, short-retention, auto-expiring — not a media library.

**MVP sequencing:** build 5.1 first and gate the first DIRECT evaluation on it.
Reference (5.2) is added only if it does not block that evaluation (draft §26).

---

## 6. Model Gateway structured-output contract (tightened)

Replaces the untyped stub. The gateway is the ONLY module that knows a provider.

```python
# packages/model_gateway — proposed typed surface (design; not yet implemented)
class StructuredRequest:
    task: str                 # e.g. "creative_intent.resolve" (domain task, NOT a model id)
    schema: dict              # JSON Schema the output MUST satisfy
    prompt_version: str       # e.g. "creative_intent/v1"
    inputs: dict              # task inputs (user text, template priors, context)
    max_repair_attempts: int = 1

class StructuredResult:
    data: dict                # schema-valid payload
    usage: Usage | None       # tokens/cost when available (logged, not domain)
    attempts: int
    repaired: bool

class ModelGateway(Protocol):
    def generate_structured(self, req: StructuredRequest) -> StructuredResult: ...
```

Behavior the gateway guarantees (so resolver + gateway can't be built
incompatibly):
1. Enforce the provided JSON Schema on model output.
2. On invalid output: perform a **bounded** structured-output repair/retry
   (`max_repair_attempts`), re-validate.
3. If still invalid: raise a typed `StructuredOutputError` — the resolver then
   returns a controlled failure/safe fallback (§9). **Never silently invent
   required values.**
4. Select the concrete model from **configuration**, keyed by `task`. No provider
   id ever reaches domain code or the contract.
5. Emit observability (§8) — never the domain object's concern.

Latency/cost: the resolver task needs strong NL interpretation, reliable
structured output, low/moderate cost, low latency. Reference analysis
additionally needs multimodal. Concrete model ids are config, chosen at
deploy time; CreativeIntent is provider-neutral.

---

## 7. Source precedence (deterministic combine)

When sources conflict, explicit user intent wins (draft §5):
```
explicit user text  >  explicit user modifications  >  reference-derived
                    >  template defaults  >  system/product defaults
```
Example: template `cinematic/slow` + user "keep the look but make it fast" →
`style_tags` keep cinematic, `pace = fast`. This precedence is applied
deterministically where sources are structured (template priors, explicit
settings); the model resolves only the free-form language and ambiguous
combinations.

---

## 8. Observability (outside the domain object)

Record per resolution, in logs — never in `CreativeIntent`:
`operation`, `model`, `provider`, `prompt_version`, `latency_ms`, `attempts`,
`structured_output_ok`, `usage{input,output}` when available, `correlation_id`,
`input_source_types`. Do not log raw user media or sensitive content. This
reuses the structured-log field shape in 03 §37a.

---

## 9. Failure behavior

CreativeIntent failure must never corrupt Story state (draft §19).
- Reference analysis fails but text present → continue with text (degrade).
- Model unavailable / network down (server) → return a recoverable UI state;
  the device MAY fall back to the local mock intent, **flagged degraded**, so the
  user is never blocked. The degraded intent is replaced on next successful
  server resolution.
- Structured output invalid after bounded repair → typed error → safe fallback,
  not a silently-fabricated intent.

---

## 10. Prompts & evaluation (net-new infra)

- **Prompts** externalized + versioned: `prompts/creative_intent/v1/{system.md,
  examples.json, schema.json}` + a `current` pointer. Prompt version is logged
  and associated with eval runs (draft §22).
- **Eval fixtures** under `evals/creative-intent/`: the draft §24 cases (explicit,
  emotional, relative language, subject emphasis, negative preference, minimal
  input, template+override, reference+override, conflicting inputs). Evaluate
  per §25 criteria; the decisive question is whether Director can propose a good
  StoryConcept from `CreativeIntent + independent SceneUnderstanding`.
- Creative Intent is evaluated **independently before** DIRECT integration.

---

## 11. iOS changes required (Milestone C client work)

Currently iOS resolves intent locally and never calls the API. For this design:
1. Add a networking client (`Data/API/`) that calls `POST /v1/creative-intent`
   and decodes `CreativeIntentContractDTO` (already defined, snake_case
   CodingKeys, ISO-8601).
2. Map the returned contract into the SwiftData `StoryModel.intent*` columns
   (flattened persistence; mapping documented in `02 §0a`). The device model
   does not need every contract field persisted for MVP — persist what the UI
   and Create flow use; keep the full contract available for the Director call.
3. Keep the local Swift `MockStoryEngine` ONLY as the offline/degraded path (§9).
4. Reference flow (5.2) additionally needs the presigned-upload client.

---

## 12. Definition of Done (Creative Intent ready for DIRECT)

Per draft §28, concretely:
- [x] canonical `CreativeIntent` contract frozen (incl. `pace`,
      `creative_constraints`) — DONE this milestone (schema + Pydantic + Swift +
      tests, 38 passed).
- [ ] text/template resolution reliable (server, model-backed).
- [ ] template+text precedence works.
- [ ] structured output validates reliably; bounded repair in gateway.
- [ ] physical-scene hallucinations rare (eval).
- [ ] meaningful user constraints survive normalization (eval).
- [ ] output compact; Director consumes it without raw prose for ordinary cases.
- [ ] prompt versioning exists; eval fixtures exist.
- [ ] latency + approximate cost measured.
- [ ] failure behavior implemented (§9).

At that point CreativeIntent becomes a real upstream dependency of Story Director.

---

## 13. Open questions to resolve by experiment (not pre-frozen)

From draft §27, still open and cheap to test:
1. `desired_feeling` controlled vocabulary vs free-form (currently free-form).
2. Whether `emphasis` earns its place after Director evals.
3. Whether reference analysis should be its own persisted structured object
   before normalization (recommend: yes, a transient `ReferenceCharacteristics`,
   not persisted in MVP).
4. Whether a small/fast model suffices for text-only, or Director's model is
   needed for quality.
5. Which fields Director actually consumes — prune speculative fields based on
   demonstrated need, not up front.

Do not freeze these answers until the Director eval provides evidence.
