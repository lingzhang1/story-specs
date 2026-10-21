# B0 Phase 2 — Increment 1: Dedicated YoTale Management Account (Runbook)

**Scope:** create + secure ONE standalone AWS account (root `aws@yotale.ai`) and make it the
workload-free management account of a NEW AWS Organization. Configure Identity Center, alternate
contacts, root-email + recovery, MFA. **No member accounts, no app infra, no DNS/SES/CI/CD changes.**

**Separate from existing:** account `989088054552` / org `o-hwquoahv3t` (1 account, master
`dearlingzhang@gmail.com`) is UNTOUCHED. The new account forms its own org as a new payer.

**Agent boundaries honored:** the agent never requests or stores the root password, MFA codes/secret,
or payment credentials. Console steps are owner-performed; the agent verifies via read-only signals.

---

## Step 1 — Create the standalone AWS account (console, owner)
1. Open an incognito/private browser window (so it doesn't reuse the dev-account session).
2. Go to the AWS sign-up page: https://signup.aws.amazon.com/
3. **Root email:** `aws@yotale.ai`  ·  **Account name:** `YoTale-Management` (or similar).
4. Set a strong **root password** (store in YOUR password manager — never share it with me).
5. Contact info: your business/personal details.
6. **Payment method:** a valid card (AWS places a temporary ~$1 auth, refunded). Required even though
   the management account will run no workloads.
7. **Phone verification:** enter the code AWS sends (SMS/voice) — on your phone only.
8. **Support plan:** choose **Basic (free)**.
9. Finish. AWS emails a confirmation to `aws@yotale.ai` → it forwards to your Gmail.

**Verify (owner → tell me):** you received the AWS welcome/confirmation email at your Gmail
(via the `aws@` forward). This re-proves the forward on a real AWS message.

## Step 2 — Secure the ROOT user immediately (console, owner)
1. Sign in as **root** (the `aws@yotale.ai` email).
2. **Enable MFA on root** (IAM → Security credentials, or the sign-in security panel). Prefer a
   hardware key or an authenticator app. Store backup codes in your password manager.
3. Do **not** create root access keys (AWS discourages this; leave none).
4. Account → **Alternate contacts**: set **Billing**, **Operations**, **Security** contacts
   (use `aws-billing@`, `alerts@`/ops, `aws-security@` — all forward to your Gmail).
5. Account settings: confirm the **root email** and a recovery phone are correct.

**Verify (owner → tell me):** root MFA shows "Enabled"; alternate contacts saved.

## Step 3 — Enable AWS Organizations with ALL features (console, owner)
1. As root (or an admin), open **AWS Organizations** → **Create an organization**.
2. Choose **Enable all features** (FeatureSet = ALL) — NOT consolidated-billing-only.
3. This account becomes the **management account** of a brand-new org (new `o-xxxx` id).
4. Do **not** create any member accounts yet.

**Verify (agent, if granted read):** `describe-organization` shows a NEW org id ≠ `o-hwquoahv3t`,
FeatureSet=ALL, MasterAccountId = the new account, MasterAccountEmail = `aws@yotale.ai`.

## Step 4 — IAM Identity Center + administrative access (console, owner)
1. Set region to **us-east-1** (keep the org/identity home region consistent).
2. Open **IAM Identity Center** → **Enable**. (Choose Identity Center directory as the identity source
   to start; external IdP can come later.)
3. Create an **admin user** (your email) and a **permission set** (`AdministratorAccess` to start;
   tighten to least-privilege for deploys in Phase 3).
4. Assign the admin user to the management account via that permission set.
5. Use the Identity Center **access portal URL** for day-to-day admin — stop using root.

**Verify (owner → tell me):** you can sign in through the Identity Center portal with the admin
permission set (no longer using root for daily work).

## Step 5 — (Optional) grant the agent a read-only verification role
To let me verify org/account config directly, create in the NEW management account an IAM role or
Identity Center permission set with **`ReadOnlyAccess`** (or `arn:aws:iam::aws:policy/job-function/ViewOnlyAccess`)
and give me a way to assume it (profile). I will NEVER need write access, the root user, or keys.
If you skip this, I verify from your console confirmations + public DNS/billing signals instead.

---

## STOP gate (end of Increment 1)
After Steps 1–4 verify, STOP. Report: new Account ID, new Org ID, FeatureSet, Identity Center status,
alternate contacts, root MFA, and that `989088054552`/`o-hwquoahv3t`/dev/Ainigama are untouched.

## Costs
- Management account itself: **$0** base (no workloads). Temporary ~$1 card auth at signup (refunded).
- Organizations, Identity Center (Identity Center directory), IAM: **$0**.
- No member accounts / no infra this increment → no recurring charge.

## Prerequisites for member-account creation (next increment, NOT now)
- Confirm root emails `beta@`/`gamma@`/`prod@` receive mail (same forward).
- Decide OU layout (Workloads OU: beta/gamma/prod; Security/Sandbox OUs).
- Draft SCP guardrails (region lock us-east-1 + exceptions; deny leaving org; protect security infra).
- Per-account budgets ($100/$50/$100 @ 50/80/100%).
- Owner go for member-account creation.
