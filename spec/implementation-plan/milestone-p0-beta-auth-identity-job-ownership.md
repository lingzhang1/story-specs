# Milestone — P0 Beta Authentication, Identity & Job Ownership (DESIGN + PROPOSAL)

**Status:** DESIGN + PROPOSAL (2026-10-18). Read-only audit + architecture only. **No code changed, NO AWS
resources provisioned, nothing deployed, no billing built.** **STOP at the authentication-architecture
approval gate.** Supersedes nothing; CI/CD Gate 2 is CLOSED and not reopened here.

Owner decisions driving this: unified account with a stable internal `user_id`; two login methods (Sign in
with Apple + passwordless Email OTP) mapping to ONE account model; per-user job ownership; account linking +
recovery; beta allowlist; layered privacy-conscious abuse prevention; quota/billing *foundation* only.

---

## Phase 0 — Audit findings (read-only, evidence-cited)

### A. Backend authentication — **there is none on the paid AI surface**
- **No auth middleware, no JWT, no `Authorization` header, no CORS config** anywhere in `apps/api/app/`.
  Every FastAPI `Depends(...)` is an **engine injector**, not an auth dependency.
- The ONLY gate is `_dev_access_ok()` (`apps/api/app/main.py:992`), which returns
  `os.environ.get("STORY_JOBS_API_ENABLED","1") == "1"` — **default OPEN**. Its own comment:
  *"UUIDv5 secrecy is NOT authentication … a real auth principal replaces this when auth lands."*
  It guards ONLY the 7 async job routes.
- `owner` is a **client-supplied form field, default `"dev"`, explicitly "NOT trusted as identity"**
  (`main.py:1001,1114`).

### B. Async job ownership — **cross-user read/cancel is possible today**
- `job_id = uuid5(_JOB_NS, operation_id)` (`packages/edit_plan_jobs/__init__.py:61-63`) — **deterministic &
  guessable** from the operation id.
- `GET /v1/edit-plan-jobs/{job_id}` (`main.py:1073`) and `/cancel` (`main.py:1086`) check only
  `_dev_access_ok()` + existence — **no ownership check**. Any caller who knows/guesses a `job_id` can read
  its plan or cancel it. Same for the coverage/v2 job routes.
- **Idempotency + lease fencing are SOLID and must be preserved:** `payload_fingerprint` → HTTP 409 on a
  same-op different-payload submit; `_set_status(..., lease_token=...)` conditional writes fence a stale
  worker (`packages/edit_plan_jobs/__init__.py:200,299-322`). DynamoDB item already carries `owner`,
  `operation_id`, `lease_until`, `lease_token`.

### C. Synchronous AI endpoints — **only 2 of ~15 AI operations are async; the rest run in-process, all open**
Every model-backed engine calls `BedrockModelGateway` (boto3 Bedrock Converse). Topology verified:

| Endpoint | Handler:line | AI work | sync/async | Auth today |
|---|---|---|---|---|
| `POST /v1/creative-intent` | 371 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/interpret-revision` | 420 | Bedrock, up to 2 calls | **sync** | **none** |
| `POST /v1/scene-understanding` | 481 | Bedrock vision (Nova Pro) | **sync** | **none** |
| `POST /v1/asset-understanding` | 562 | Bedrock vision (Sonnet), 24 frames | **sync** (sem=4) | **none** |
| `POST /v1/story-coverage` | 681 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/story-guide` | 720 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/edit-plan` | 775 | Bedrock (Opus→Sonnet) + ≤2 repair | **sync** | **none** |
| `POST /v1/edit-plan-mm` | 879 | Bedrock multimodal + ≤2 critique, ≤20MB img | **sync** | **none** |
| `POST /v1/edit-plan-jobs` (+status/cancel/reconcile) | 999… | SQS → Lambda worker | submit sync, **exec async** | `_dev_access_ok` (open) |
| `POST /v1/story-coverage/v2/jobs` (+status/cancel) | 1112… | SQS → Lambda worker | submit sync, **exec async** | `_dev_access_ok` (open) |
| `POST /v1/music-plan` | 1191 | Bedrock (Opus→Sonnet) | **sync** | **none** |
| `POST /v1/duration-recommendation` | 1231 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/story-coverage/v2` | 1441 | **2× Opus** (~191s) | **sync** | **none** |
| `POST /v1/director/initialize` | 1498 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/director/next` | 1517 | Bedrock (Opus) | **sync** | **none** |
| `POST /v1/music/{candidates,search,analysis,soundtrack}` | 1277… | **Epidemic HTTP** (paid 3rd-party, not Bedrock) | sync | global `STORY_EPIDEMIC_MODE` flag only |
| `POST /v1/music-direction`, `/v1/stories`, `/v1/stories/{id}/outputs`, `/health`, all job GET/cancel | — | deterministic / mock / read | sync | none needed |

**Takeaway: protecting only Edit Planning is insufficient** — 13 sync Bedrock endpoints + 4 paid Epidemic
endpoints are all open. Highest blast radius: `story-coverage/v2` (2× Opus), `edit-plan-mm` (multi-call +
20MB), the two vision endpoints.

### D. iOS integration — local-first, no auth, one clean seam
- Root scene is `WindowGroup { HomeView() }` (`App/StoryApp.swift:60`) — **no login gate, no onboarding.**
- **Single network seam:** `enum StoryBackend` (`Data/Contracts/CreativeIntentService.swift:15-73`) holds the
  base URL (`STORY_API_BASE_URL` env → dev App Runner) + 3 shared `URLSession`s. Every one of the 14
  request-builders across 11 service files sets **only `Content-Type`** — **no `Authorization` anywhere.**
- SwiftData: no `user_id` on any `@Model`; one **unused** opaque `LocalIdentityModel`
  (`Data/Persistence/Models.swift:458`). One wire field models ownership: `owner:"dev"`
  (`EditPlanJobClient.swift:81`, `StoryCreationCoordinator.swift:576`).
- **No session/logout** exists; only operation-recovery (`PendingOperationRecovery`, AU drain) on
  launch/foreground.
- **Smallest non-destructive integration:** store the account on `LocalIdentityModel` (additive fields),
  **adopt existing local Stories/Moments on first sign-in with zero data movement**, inject the token + real
  `owner` at the single `StoryBackend` seam. No per-record migration, no destructive step.

### E. AWS identity infrastructure — none for Story; SES in sandbox
- **No Story Cognito pool.** (The account has `ainigama-backend-{dev,prod}-user-pool` — a *different*
  project, not Story.) A CDK doc-comment references "Cognito … Phase A" as *planned* but it was never built.
- **SES has 0 verified identities and `ProductionAccessEnabled=False` → SES SANDBOX** (can only send to
  verified addresses). Real dependency for Email OTP delivery.
- Existing: DynamoDB jobs table, SQS + DLQ, Lambda worker, App Runner, ECR — all in dev acct 989088054552.

---

## Phase 1 — Authentication architecture proposal

### Provider recommendation: **Amazon Cognito User Pool (one pool, the single IdP)**
Compared against building custom + against Auth0/Firebase:

| Criterion | **Cognito (recommended)** | Custom (DIY OTP + Apple verify) | Auth0 / Firebase |
|---|---|---|---|
| Sign in with Apple | Native federated IdP | Hand-verify Apple JWT (JWKS, nonce, revocation) | Native |
| Passwordless Email OTP | **Native** (custom-auth Lambda triggers: Define/Create/Verify) or built-in email OTP | Build issuance/expiry/attempt-limit/cooldown + email yourself | Native |
| Stable internal user_id | **Pool `sub` (UUID), immutable** | you own it | vendor `sub` |
| One account, many identities | `AdminLinkProviderForUser` merges federated → one user | you build linking | supported |
| Cost at beta scale | Free ≤ 50k MAU, then ~$0.0055/MAU | $0 vendor but high eng + ops | billed per MAU sooner |
| AWS-native (IAM, SES, CloudWatch) | **First-class** | — | external |
| Token format | Standard OIDC JWT (RS256, JWKS) | roll your own | OIDC JWT |

**Why Cognito:** it is the only option that natively covers **both** Apple federation **and** passwordless
Email OTP under **one** user pool with **one** stable `sub`, stays inside the existing AWS account/IAM/SES/
CloudWatch stack, and is free at beta scale. A custom system re-implements exactly the security-critical
primitives (OTP single-use/expiry/rate-limit, Apple JWT verification, token signing/rotation) that are easy
to get subtly wrong. **Recommend ONE provider architecture, not two.**

### 1. Unified account model (conceptual — Cognito-backed, with a thin app-owned profile table)
Cognito owns authentication; a small DynamoDB table owns Story-specific account state so `user_id` is stable
and decoupled from any provider detail.

```
StoryUser            (DynamoDB: pk = user_id)
  user_id            = Cognito pool `sub` (UUID) — THE authoritative identity
  created_at, account_status (active|suspended|deleted)
  beta_access_status (invited|active|denied)

AuthIdentity         (managed INSIDE Cognito; mirrored read-only if needed)
  provider: apple | email   provider_subject   verified_email?   verified_at
  → all federated/native identities link to ONE Cognito user = one user_id

UserUsage            (DynamoDB: pk = user_id, sk = quota_period)
  generation_usage, quota_period, entitlement_status   (Phase 4)
```
`user_id` is NEVER an email, Apple relay address, or device id. Email/Apple-subject are attributes of an
`AuthIdentity`, not the primary key.

### 2. Sign in with Apple
- iOS `ASAuthorizationController` → Apple identity token → Cognito Apple IdP (federated).
- **Server trusts only the Cognito-issued JWT**, whose `sub` = `user_id`. Apple's `sub` is stored as the
  `AuthIdentity.provider_subject`; **Apple email (incl. private relay `@privaterelay.appleid.com`) is NEVER
  the identity** — stored as an optional contact attribute only.
- Refresh via Cognito refresh token (keychain-stored); revoked-credential handling via Apple's
  server-to-server notifications → disable the linked identity (not the whole account unless last identity).

### 3. Email OTP (passwordless)
- Cognito **custom-auth flow** (Define/Create/Verify Lambda triggers): 6-digit code, **5-min expiry,
  single-use, ≤5 attempts, 60s resend cooldown**, per-email + per-IP issuance rate limit.
- **Delivery:** Cognito → SES. **Dependency: SES is in sandbox** — must request production access (or verify
  a sending domain) before external beta; until then OTP works only to verified addresses.
- No passwords, ever. Session issuance = same Cognito JWT/refresh as Apple → identical downstream path.

### 4. Account linking
- **Explicit user action only**, with **proof of control** over the new identity (complete a fresh sign-in /
  OTP on the method being linked), then `AdminLinkProviderForUser` → both identities resolve to the same
  `user_id`. **No automatic merge on matching email.** No silent merge of two established accounts (if both
  already have content/entitlements → surface a manual, owner-reviewed merge, out of beta scope).
- Private-relay-safe: linking keys on verified provider subject + the user's explicit action, never on email
  string equality. Recovery: a user who loses one method signs in with the other; losing all methods →
  support path (documented, not automated).

---

## Phase 2 — Device identity & abuse prevention

### 1. Device-signal feasibility (what each CAN and CANNOT establish)
| Mechanism | Establishes | Does NOT establish | Use |
|---|---|---|---|
| **App Attest** | this is a genuine, unmodified build of OUR app on a real Apple device (hardware-backed key) | a *person*; is not a stable cross-reinstall id | gate who may *request* introductory credit |
| **DeviceCheck** | 2 bits of per-device state that **persist across reinstall** (per developer acct) | identity; survives device wipe/replacement | mark "this device already claimed intro credit" |
| **Keychain device credential** | app-generated key, survives reinstall (until keychain cleared) | hardware identity; user clears it | soft signal only |
| **IDFV** | same-vendor install grouping | cross-vendor; resets when all our apps removed | weak signal only |

**Never collect** IMEI, serial, hardware UDID, or prohibited fingerprints. **One device ≠ one person.**

### 2. Free-credit protection — layered
- **Account level:** each verified Story account gets a bounded introductory AI allowance, subject to beta
  eligibility. (The real cap lives server-side, Phase 4.)
- **Device level (where supported):** DeviceCheck's 2 persistent bits mark a device that already consumed
  intro credit — *reduces* repeat claims, never a hard ban.
- **Backend level:** quotas + concurrency enforced on the **trusted `user_id`** — the authoritative control.
- **Beta access:** invite allowlist, independent of creating a new email — the primary gate against
  register-N-emails abuse.
- **Abuse monitoring:** low-cardinality metrics on repeated signups / abnormal quota burn / unusual inference
  volume, **without** logging emails/content.

### 3. Legitimate-user safeguards (false-positive protection)
Shared iPhone (family), one user with multiple iPhones, reinstall, device replacement, Apple+email on one
account, relay-email change, DeviceCheck unavailable/failing, travel/network change — **none may cause an
irreversible ban based on device signals alone.** Device signals only *soften* intro-credit grants; the
account + beta allowlist are authoritative. Document privacy + false-positive risk. **No fraud-scoring system
for beta.**

---

## Phase 3 — Backend authorization & ownership (minimum secure design)
- **Trusted identity:** a FastAPI dependency `require_user()` validates the Cognito JWT (RS256 via cached
  JWKS, `iss`/`aud`/`exp` checks) and yields `user_id = token.sub`. **Every protected request derives
  `user_id` server-side; the client `owner` field is dropped/ignored** (kept wire-compatible but overwritten).
- **Job ownership:** on submit, bind the job to the authenticated `user_id` (store in the existing `owner`
  attribute — now trusted). Enforce ownership on **submit/status/cancel** via a DynamoDB condition / post-read
  check (`job.owner == user_id` else **404**, not 403 — no cross-user existence disclosure). **Scope the
  operation_id / idempotency to the principal** (fingerprint key includes `user_id`) so two users cannot
  collide on one `operation_id`. **Lease fencing + payload-conflict 409 unchanged.**
- **Expensive-endpoint protection:** apply `require_user()` to **all** Bedrock + Epidemic endpoints in the
  table above (not just edit-plan). Additionally rate-limit + concurrency-cap + quota-check the heavy ones
  (coverage/v2, edit-plan*, director*, vision, music/*).
- **Local-first compatibility:** opening Story, capturing Moments, browsing the local library, and viewing
  locally-saved Stories **require NO sign-in**. Auth is required only before a **cloud AI** call. On first
  sign-in, local content is *adopted* by the account (set `LocalIdentityModel.accountId`, zero data movement);
  **never delete/recreate SwiftData.** Logout clears tokens but **leaves local content on device**; account
  switch re-scopes new cloud calls but does not move existing local Stories. **Legacy dev jobs** (owner="dev",
  pre-auth) are treated as orphaned/dev-only: not readable by any real `user_id`, retained, never served to a
  beta account.

---

## Phase 4 — Quota & billing FOUNDATION (design only; no billing built)
- **Server-side usage accounting** (`UserUsage`): count AI generations, revisions, Create-Another-Version,
  shot guidance, music planning, coverage, other paid inference. **Track actual model usage + estimated cost
  separately from user-visible credits** (reuse the observability cost estimator).
- **Quota enforcement (proposed):** per-user allowance; concurrent-job limit; rate limit; **atomic quota
  reservation BEFORE expensive inference** (DynamoDB conditional update / counter), **idempotent retry**
  (same operation_id reserves once), **release/refund on failure/cancel**, guard against concurrent bypass
  (conditional writes, not read-modify-write). **Backend is authoritative; client counters display-only.**
- **Future subscriptions:** plan for StoreKit 2 + **server-side receipt/transaction validation**;
  entitlements belong to the **stable Story account**, not a device/email. **Not implemented this milestone.**

---

## Phase 5 — Security, privacy & observability (extends the beta-observability design)
- **Metrics (low-cardinality dims):** sign-in success/fail (by provider), OTP requests + verify failures,
  unauthorized-access attempts, cross-user job-access denials, beta-allowlist denials, quota denials,
  quota-enforcement errors, suspicious-signup signal, AI usage by workload, auth-service errors.
- **NEVER log:** OTP codes, tokens, raw credentials, full emails (hash or domain-only), precise GPS, private
  user content.
- **Retention/deletion:** define token/session TTLs (short access token, rotating refresh), and an account
  deletion path (Cognito user + `StoryUser`/`UserUsage` purge, jobs anonymized/retained per policy). Device
  signals must **not** be the sole basis for billing, identity, or irreversible enforcement.

---

## Phase 6 — Implementation plan & acceptance

### Increments (each: files, infra, risks, tests, device tests, effort, rollback)
1. **IdP selection + Cognito pool (CDK)** — new `story-auth-stack` (user pool, Apple IdP, custom-auth OTP
   Lambda triggers, app client, SES wiring). Infra: Cognito + 3 Lambdas + SES prod-access request. Risk:
   SES sandbox. Tests: OTP issue/verify/expiry/attempt-limit unit; Apple-token verify. Device: real Apple
   sign-in. Rollback: stack is additive; app runs unauthed until the seam flips. **~2 days.**
2. **Backend auth middleware** — `require_user()` dependency + JWKS cache (`apps/api/app/auth.py` new);
   wire into app. Risk: clock-skew/JWKS rotation. Tests: valid/expired/wrong-aud/tampered JWT. **~1.5 days.**
3. **Job ownership enforcement** — bind `user_id`, ownership checks on submit/status/cancel, principal-scoped
   fingerprint (`main.py`, `packages/edit_plan_jobs`). Preserve lease + 409. Tests: cross-user 404,
   idempotency collision across users, lease unchanged. **~1.5 days.**
4. **Protect sync AI endpoints** — apply `require_user()` to all 13 Bedrock + 4 Epidemic routes; add
   rate/concurrency on heavy ones. Tests: 401 unauth on each; authed passes. **~1 day.**
5. **iOS Apple sign-in** — `SignInView`, `AuthStore` (Cognito SDK / ASWebAuth), token in keychain, inject
   `Authorization` at `StoryBackend` seam, `owner` = real `user_id`. **~2 days.**
6. **iOS Email OTP** — OTP entry UI + resend cooldown; same `AuthStore`. **~1.5 days.**
7. **Account linking + session recovery** — link UI (explicit + proof), refresh/restore, revoked-cred
   handling. **~2 days.**
8. **Beta allowlist** — invite list check at sign-in/first-use (Cognito group or `StoryUser.beta_access`).
   **~0.5 day.**
9. **Device-abuse safeguards** — App Attest/DeviceCheck integration + intro-credit soft-limit. **~2 days.**
10. **Quota integration points** — reservation/release wiring at expensive endpoints (foundation). **~2 days.**
11. **CloudWatch security telemetry** — the Phase 5 metric set. **~1 day.**
12. **Local-content adoption** — `LocalIdentityModel.accountId`, adopt-on-sign-in, logout semantics. **~1 day.**

**Estimate: ~19–20 eng-days** (several blocked on SES production access + the prod account; iOS needs a real
device + Apple Developer config).

### Beta security acceptance matrix (required-before-beta marked ✱)
| Scenario | Req. before beta |
|---|---|
| Apple sign-in; Email OTP sign-in | ✱ |
| Expired / incorrect / reused OTP rejected | ✱ |
| Link Apple + email → one account | ✱ |
| Two accounts, different emails, same device | ✱ |
| One account on multiple devices | ✱ |
| Shared-device behavior (no false ban) | ✱ |
| Session expiration + refresh/recovery | ✱ |
| Unauthorized job status/cancel → 404 | ✱ |
| Cross-user idempotency collision isolated | ✱ |
| Concurrent quota consumption (no bypass) | ✱ |
| Reinstall / device replacement (no lockout) | ✱ |
| Offline capture without login | ✱ |
| Existing local Stories survive sign-in/logout | ✱ |
| Beta-allowlist denial | ✱ |
| Invalid App Attest/DeviceCheck handled gracefully | later |
| Auth + quota failures appear in CloudWatch | ✱ |

---

## Remaining risks & owner decisions
1. **SES sandbox** — Email OTP cannot reach arbitrary addresses until production access is granted (owner
   action). Decision: request SES prod access, or restrict OTP to a verified domain for closed beta.
2. **Prod account** — this design provisions in dev only; prod Cognito/SES need the separate prod account.
3. **Apple Developer config** — Sign in with Apple needs a Services ID, key, and the app's entitlement
   (owner/Apple-portal action).
4. **Managed vs custom confirmation** — recommend Cognito; confirm before building `story-auth-stack`.
5. **Legacy dev-job policy** — proposed: retain, never serve to real accounts. Confirm.
6. **STOP GATE honored:** no AWS identity resources provisioned, no auth deployed, no billing, CI/CD not
   reopened, no unrelated features. Awaiting architecture approval.
