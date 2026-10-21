# YoTale B0 — AWS Multi-Account & Official Email Foundation (PLAN + AUDIT)

**Status:** PLAN / PROPOSAL — read-only audit complete. NOTHING provisioned, NO DNS changed, NO account
created, NO mailbox created, NO beta deployed. STOP at owner approval. Date: 2026-10-21. Domain: `yotale.ai`
(Namecheap). Account under audit: `989088054552`.

---

## Phase 0 — Read-only audit (verified facts)

### Accounts & Organization
- The current dev account **`989088054552` IS ALREADY an Organization MANAGEMENT (master) account** —
  org `o-hwquoahv3t`, FeatureSet **ALL**, master email **`dearlingzhang@gmail.com`**, SCPs ENABLED.
- The org is **flat**: **no OUs**, only the AWS-managed `FullAWSAccess` SCP (no custom guardrails).
- It is the **ONLY account** in the org (no beta/gamma/prod exist yet).
- **IAM Identity Center is ACTIVE** (instance `ssoins-72234f2ceb354089`, identity store `d-906627f3b6`) in
  this account — a centralized-access foundation already exists.
- **This management account also runs WORKLOADS for MULTIPLE projects** — not only Story
  (`story-development-*`) but a second app `ainigama-backend-dev-*` (its own DynamoDB tables + Cognito user
  pool `us-east-1_s7Sb0KzOc`). This is the anti-pattern the owner flagged: the management account hosts
  application workloads, and two unrelated projects share one account.

### Dev resources (Story, us-east-1)
- DynamoDB: `story-development-stories`, `story-development-understanding`, `story-development-media`,
  `story-development-edit-plan-jobs`, identity/claims/users tables.
- S3: `story-development-media-…`, `story-development-edit-plan-staging-…`, `story-codebuild-src-…`,
  `story-assetunderstanding-exp-…`.
- Lambda: `story-development-edit-plan-worker` (+ CDK custom-resource lambdas). App Runner
  `story-development-api`. SQS `…-edit-plan-jobs` + `…-edit-plan-jobs-dlq`. Cognito `us-east-1_AzfHOEdA5`.

### Email — DNS & SES (authoritative via live `dig` + SES API)
- **Nameservers: `dns1/dns2.registrar-servers.com`** → DNS is **Namecheap BasicDNS** (NOT Route 53). All DNS
  changes happen in the Namecheap dashboard.
- **SES SENDING is fully working and MUST be preserved:**
  - `yotale.ai` domain identity **verified**, **SendingEnabled**.
  - **DKIM Success** — 3 CNAMEs confirmed LIVE in Namecheap
    (`{qlfked64…,kmlu75uj…,nljlpzwc…}._domainkey.yotale.ai` → `*.dkim.amazonses.com`).
  - **Custom MAIL FROM `mail.yotale.ai` Success** — live `MX 10 feedback-smtp.us-east-1.amazonses.com` +
    `TXT v=spf1 include:amazonses.com ~all` on that subdomain.
  - **Cognito OTP send policy** bound to user pool `us-east-1_AzfHOEdA5` (`AllowCognitoToSendFromYoTaleIdentity`).
  - **DMARC present at apex:** `_dmarc.yotale.ai TXT v=DMARC1; p=none`.
- **THE GAP (what blocks account registration):** the **apex `yotale.ai` has NO MX record and no apex SPF**,
  so **inbound mail to any `@yotale.ai` address is NOT received anywhere today** — every proposed address
  (aws@, beta@, …) would currently **bounce**. This is exactly what must be fixed + verified BEFORE creating
  any AWS account (AWS sends verification/recovery mail to the registration address).

### Deployment pipeline & release artifacts (already sanctioned)
- `.github/workflows/deploy-dev.yml`: **keyless GitHub OIDC** (`DEV_DEPLOY_ROLE_ARN`), protected
  `development` GitHub Environment, manual-dispatch only, concurrency-guarded, **dev account only**.
- **Immutable artifacts are ALREADY the release authority:** API image tagged `git-<sha>` and **deployed by
  DIGEST** (`cdk deploy -c imageDigest=…`); worker built by `scripts/build_worker_asset.sh` into a hashed
  zip; a release manifest is written per deploy. CFN driven through a **scoped cfn-exec role**
  (`CFN_EXEC_ROLE_ARN`), not admin. → The owner's "no mutable `:latest` as release authority" is **already
  satisfied by the sanctioned path**; the `:latest`/CodeBuild route used for dev iteration is the documented
  out-of-band deviation, NOT the beta release path.
- CDK is **already environment-parameterized**: `bin/app.ts` reads `-c environment=` and
  `CDK_DEFAULT_ACCOUNT/REGION`; the auth stack is gated behind `-c deployAuth=true`. Beta/gamma/prod are a
  matter of context + a target account + a bootstrapped OIDC/exec role — not a rewrite.

---

## Phase 1 — Email-receiving proposal (smallest reliable setup)

**Recommendation: Namecheap Email Forwarding for inbound**, because (a) DNS already lives at Namecheap, (b) it
is free with the domain, (c) it adds ONLY an apex `MX` + does not touch any SES sending record, and (d) it
forwards AWS registration/verification/security/recovery mail to a mailbox the owner already controls. SES
*receipt* (the heavier alternative) is unnecessary for a handful of account addresses and would require
Route 53 or careful MX coexistence. Revisit SES-receipt/Workmail only if shared team inboxes are needed later.

### Exact DNS changes to make in Namecheap (apply only after approval — SHOWN, not applied)

ADD (apex receiving — Namecheap Email Forwarding auto-manages these when forwarding is enabled; values shown
for verification):
```
Type  Host  Value                              Priority  TTL
MX    @     mx1.privateemail.com               10        Automatic   # (Namecheap forwarding MX)
MX    @     mx2.privateemail.com               10        Automatic
TXT   @     v=spf1 include:spf.efwd.registrar-servers.com ~all   Automatic   # apex SPF for forwarding
```
NOTE: the exact forwarding MX hostnames are shown by Namecheap when Email Forwarding is toggled on; the two
above are Namecheap's current forwarding endpoints — confirm in the dashboard before saving.

DO NOT TOUCH (sending — leave exactly as-is, verified live):
```
CNAME  qlfked64yn7gvnozgzlvfylbldlje4ds._domainkey  → …dkim.amazonses.com
CNAME  kmlu75uj6qmy6iwkso5reeilhhvgfcx5._domainkey  → …dkim.amazonses.com
CNAME  nljlpzwcdfjkqjdk6fx6i3553tahg62u._domainkey  → …dkim.amazonses.com
MX     mail (mail.yotale.ai)  10 feedback-smtp.us-east-1.amazonses.com
TXT    mail (mail.yotale.ai)  v=spf1 include:amazonses.com ~all
TXT    _dmarc  v=DMARC1; p=none
```
CONFLICT CHECK: SES MAIL FROM uses the **`mail.` subdomain** MX; the new receiving MX is at the **apex** —
they do not collide. The apex SPF is NEW (none exists today). Keep DMARC `p=none` during setup; tighten later.

### Forwarding map (configure in Namecheap after the MX/SPF resolve)
| Address | Forward to (owner-controlled mailbox) |
|---|---|
| aws@yotale.ai (management) | owner inbox |
| beta@yotale.ai | owner inbox |
| gamma@yotale.ai | owner inbox |
| prod@yotale.ai | owner inbox |
| aws-security@yotale.ai | owner inbox |
| aws-billing@yotale.ai | owner inbox |
| alerts@yotale.ai | owner inbox (or the ops channel) |

**Owner input needed:** the destination mailbox each address forwards to (one address or several).

### Inbound verification gate (BEFORE any AWS account creation)
For each of aws@/beta@/gamma@/prod@/aws-security@/aws-billing@/alerts@: send a test message and confirm it
arrives at the forward target. AWS account creation proceeds ONLY after all seven deliver. (A distinct,
reliably-received address per account root is an AWS best practice and a recovery safeguard.)

---

## Phase 2 — AWS Organizations architecture (smallest secure structure)

**Key decision — do NOT reuse `989088054552` as management.** It already hosts Story + ainigama workloads.
The clean target is a **dedicated, workload-free YoTale management account**. Two viable paths:

- **Path A (preferred, cleanest):** create a NEW dedicated management account under a NEW org root, with a
  workload-free management account (email `aws@yotale.ai`), and create beta/gamma/prod (and a migrated dev)
  as member accounts. The current `989088054552` org is left as-is or retired later; Story dev either stays
  there transitionally or is migrated. Cleanest separation; most setup.
- **Path B (pragmatic):** KEEP the existing org `o-hwquoahv3t` but STOP treating `989088054552` as a
  workload account going forward — add OUs, create fresh beta/gamma/prod member accounts, and migrate Story
  workloads out of the management account over time. Less upfront churn; the management account stays
  "dirty" with legacy workloads until migration. **This is a real owner decision (A vs B); flagged below.**

Target structure (either path):
```
Root
├── Management account  (email aws@yotale.ai)  — billing, Identity Center, no workloads
├── OU: Workloads
│    ├── beta     (email beta@yotale.ai)
│    ├── gamma    (email gamma@yotale.ai)
│    └── prod     (email prod@yotale.ai)
└── OU: Sandbox/Dev   (dev — current Story dev, migrated or re-created)
```
Controls:
- **Centralized billing** at management; **per-account AWS Budgets** with alerts to `aws-billing@`/`alerts@`.
- **IAM Identity Center** (already active) as the single sign-in; permission sets per account/role
  (AdminAccess scoped to break-glass; a `StoryDeploy` least-privilege set for pipelines).
- **Root security:** each account root gets a unique strong password + hardware/virtual MFA; root used only
  for break-glass; security/billing/operations alternate contacts set to aws-security@/aws-billing@.
- **SCPs on the Workloads OU:** deny leaving the org, deny disabling CloudTrail/guardrails, region-restrict
  to us-east-1, deny root-access-key creation.
- **Least-privilege cross-account deploy:** GitHub OIDC role PER environment (no static keys — the dev
  pipeline already does this); a scoped CFN exec role per account; **no persistent deployment access keys**.
- **No cross-environment data sharing by default:** separate buckets/tables/pools per account; no bucket
  policy or KMS grant spanning environments.

**Owner input needed:** Path A vs Path B; and the management-account email (`aws@yotale.ai`) must receive mail
first (Phase 1 gate).

---

## Phase 3 — Beta environment deployment plan (smallest reproducible)

Deploy the SAME CDK (`story-<env>-api` + `story-<env>-auth`) into the **beta account** with
`-c environment=beta`, by **immutable digest** (the existing pipeline pattern), into a FRESH data plane:
- Cognito user pool (beta) + account-ownership/identity tables (fresh, empty).
- `story-beta-stories` / `-understanding` / `-media` (private S3, no public access) / `-edit-plan-jobs(+dlq)`.
- App Runner `story-beta-api` + Lambda `story-beta-edit-plan-worker` (digest/hash-pinned).
- SES sending reused from the verified `yotale.ai` identity (add a beta Cognito-send policy scoped to the
  beta user pool ARN — mirrors the dev policy); OTP works day one.
- Secrets/config via SSM Parameter Store / Secrets Manager in the beta account (no secrets in the image).
- Basic CloudWatch: the free SQS/Lambda/App Runner/DynamoDB metrics + a beta budget alarm. (The full
  observability milestone — the approved next milestone — layers EMF metrics/dashboards/alarms on top.)
- **Beta cost safeguards:** per-account AWS Budget (owner amount) + Lambda reserved concurrency + Bedrock
  usage visibility (the worker now emits EstimatedModelCostUSD — commit `9d7ca94`).
- **Data isolation (hard rule):** NO copy of dev user data, personal media, or test identities into beta.
  Beta starts empty. Release authority is the **digest**, never `:latest`.

---

## Phase 4 — Deployment readiness

1. **Account-creation sequence:** (gate: Phase-1 inbound verified) → create management account (aws@) →
   enable Identity Center/OUs/SCPs → create beta (beta@), gamma (gamma@), prod (prod@) member accounts →
   set root MFA + alternate contacts on each → bootstrap CDK + OIDC deploy role in beta.
2. **Email verification actions:** enable Namecheap forwarding; add apex MX+SPF; verify all 7 addresses
   receive mail (Phase-1 gate) BEFORE step 1.
3. **Account/OU structure:** per Phase 2 (pending A/B decision).
4. **Resource inventory & cost:** beta mirrors dev (App Runner ~$5–25/mo idle-ish, DynamoDB on-demand, S3,
   Lambda, SQS, Bedrock usage-based) + CloudWatch; rough beta **< ~$50–150/mo** pre-traffic (refine on load).
5. **Beta CDK changes:** none structural — `-c environment=beta` + the beta account/region context + a
   beta Cognito-send SES policy; bootstrap the beta account.
6. **GitHub Environment/OIDC changes:** add a `beta` protected Environment with `BETA_DEPLOY_ROLE_ARN` +
   `CFN_EXEC_ROLE_ARN` vars; a `deploy-beta.yml` mirroring `deploy-dev.yml` (manual dispatch, digest-pinned).
7. **Deploy & rollback:** deploy by digest; rollback = redeploy the previous digest from the release manifest
   (immutable artifacts make this exact).
8. **Smoke test & security acceptance:** health + a real OTP sign-in in beta; private-bucket/no-public-access
   assertion; cross-account isolation check; no dev data present.
9. **Monitoring & budget:** free service metrics + beta budget alert to aws-billing@/alerts@.
10. **Remaining owner decisions:** see below.

---

## Remaining owner decisions (STOP gate — nothing proceeds without these)
1. **Org Path A (new dedicated management account/org) vs Path B (reuse org `o-hwquoahv3t`, stop using
   `989088054552` as a workload account, migrate Story out).**
2. **Forwarding destination mailbox(es)** for the 7 addresses.
3. **Confirm the 4 account root emails** (aws@/beta@/gamma@/prod@ — matches the proposal).
4. **Per-account budget amounts** (beta/gamma/prod) for the budget alarms.
5. **Region policy** (us-east-1 only, or add others) for the SCP region restriction.

Approve these and I will execute in bounded increments: Phase 1 DNS/forwarding + verification → Org/accounts →
beta bootstrap + deploy → smoke/security acceptance. The next milestone after beta acceptance is the existing
CloudWatch observability + Bedrock cost-accounting implementation.
