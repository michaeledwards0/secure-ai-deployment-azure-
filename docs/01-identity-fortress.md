<div align="center">

# Phase 1: Identity Fortress
### Zero Trust Identity Layer for AI Workload Access — Contoso AI Labs

</div>

---

## Overview

Before any AI workload existed in this environment, the identity layer had to be established — in Zero Trust architecture, identity is the perimeter. This phase builds role-based access control, a monitored break-glass account, Conditional Access (including risk-based policies), just-in-time privileged access via PIM, and a governance baseline via Azure Policy — defending against the two most common identity attack paths: **compromised credentials** and **privilege escalation**.

**Environment:** Personal Azure tenant | **Duration:** ~3-4 hours | **Standard:** SC-500 blueprint, NIST 800-207 (Zero Trust)

---

## Case Study

### Objective
Establish the Zero Trust identity layer and governance baseline for the Contoso AI Labs environment before any AI workloads are deployed, following the principle that identity is the primary security perimeter in cloud architectures — and that policy enforcement is what keeps that perimeter intact over time.

### Approach
User groups were structured around job function (Admin, Developer, End User) to enforce least privilege from the start — each role only has the access its function requires. A break-glass account was configured as a Global Admin fallback for emergencies, excluded from all Conditional Access policies so it always works if other authentication paths fail, and monitored for any sign-in activity since usage outside a true emergency should be treated as a serious alert.

Conditional Access policies were tested in report-only mode before enforcement. This validated that each policy would apply to the correct users and sign-ins without risking a lockout, flipping to enforced only after confirming expected behavior in the sign-in logs.

PIM was applied to the AI-Admins group to enable just-in-time access instead of standing privilege. Rather than holding an admin role permanently, eligible users activate it only when needed, for a limited window, with MFA and approval required. This directly reduces the blast radius of the two attack paths above — a compromised account with no standing privilege has nothing to steal.

Risk-based Conditional Access (CA05, CA06) was built directly into the Conditional Access engine rather than through the legacy Identity Protection risk policies UI, which Microsoft is retiring in October 2026. This keeps every access decision — MFA, location, and risk — in a single policy surface instead of two systems that could drift out of sync, and allows report-only validation the legacy interface doesn't support.

Azure Policy was assigned last in this phase to make the distinction between identity and governance explicit: identity controls (Conditional Access, PIM) stop unauthorized people from getting in, while Azure Policy stops authorized people from making costly configuration mistakes — like exposing a storage account to the public internet. Neither layer substitutes for the other.

### Controls Implemented
- Role-based access control via three security groups (AI-Admins, AI-Developers, AI-Users)
- MFA enforcement for all users except break-glass
- Legacy authentication blocked (eliminates a common password-spray attack vector)
- Geographic access restrictions blocking known high-risk regions
- Just-in-time privileged access via PIM with approval workflow
- Risk-based Conditional Access policies (CA05, CA06), built ahead of the legacy Identity Protection risk policies' October 2026 retirement
- Azure Policy guardrails preventing common misconfigurations at resource creation time

### Frameworks Applied
- NIST 800-207 (Zero Trust Architecture)
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 1 (Identity and Access Management)
- Microsoft Cloud Adoption Framework — Security Baseline for Identity
- Microsoft Cloud Adoption Framework — Governance disciplines

### Evidence

| Control | What it proves | Screenshot |
|---|---|---|
| Role-based groups | Three groups created mapping to distinct job functions, establishing least privilege before any user is provisioned | ![Groups](https://github.com/user-attachments/assets/8bce38ef-3870-4d8c-98f3-bc021d16159b) |
| Test identities | Three test users provisioned and assigned to their respective groups | ![Users](https://github.com/user-attachments/assets/14715a4f-9437-40c5-b83f-d0a068e75c1f) |
| Break-glass account | Emergency access account configured with Global Administrator role, excluded from all Conditional Access | ![Break-glass](https://github.com/user-attachments/assets/c2d7204f-0beb-4866-8e24-c8e8d8d25a98) |
| Named locations | Trusted and high-risk country lists defined for use in Conditional Access location conditions | ![Named locations](https://github.com/user-attachments/assets/26962d18-a958-4e50-977d-d6015842c72a) |
| Core Conditional Access policies | CA01–CA04 enforcing MFA, blocking legacy auth, blocking high-risk geographies, and requiring frequent re-auth for admins | ![CA policies](https://github.com/user-attachments/assets/efe01727-7e8a-40e1-bf22-412f0a59a2c4) |
| Just-in-time access | Sarah Admin configured as eligible (not permanently assigned) for Global Reader via PIM | ![PIM](https://github.com/user-attachments/assets/b74bb234-08e3-43d4-bffd-f604273fea1e) |
| Risk-based Conditional Access | CA05/CA06 built directly in Conditional Access, ahead of the legacy Identity Protection UI's retirement | ![Risk-based CA](https://github.com/user-attachments/assets/5a375f37-5a56-4ed7-b36e-6406697a2be3) |
| Governance baseline | Three Azure Policy assignments enforced across the subscription, protecting against public exposure and enforcing managed identity use | ![Policy baseline](https://github.com/user-attachments/assets/ec9e90d2-a110-4fa6-a798-a26f067b9a1b) |
| Enforcement confirmed live | CA01 and CA04 flipped from report-only to enforced, validated against real sign-in log results | ![Policy enforced](https://github.com/user-attachments/assets/1fcceb6f-e039-414d-83b9-5be1e612295c) |

### Lessons Learned
Privileged Identity Management proved essential for protecting against two of the most common identity attack paths: privilege escalation and compromised credentials. By requiring just-in-time activation instead of standing access, there's no permanent admin session sitting idle for an attacker to hijack.

Establishing security groups and role-based access control early was foundational to building a least-privilege environment — every subsequent decision, from Conditional Access scoping to PIM eligibility, depended on having clean, function-based groups already in place.

Policy isn't policy unless it's enforced. A control that exists only as a written rule or a report-only log entry provides visibility, not protection — the value only materializes once it's actively blocking or requiring action.

Identity controls and governance policy solve two different problems: identity controls stop unauthorized people from getting in, while Azure Policy stops authorized people from making costly mistakes. Both are necessary, and neither is a substitute for the other.

---

## Next Phase

➡️ **[Phase 2: Network Architecture & Isolation](./02-network-architecture.md)**

In Phase 2 we'll build the network layer that will house the AI workload — hub-spoke topology, private endpoints, NSGs, and Azure Bastion for admin access.

---

<details>
<summary><strong>📋 Full Execution Guide (click to expand)</strong> — step-by-step build instructions and completion checklist</summary>

<br>

**Duration:** ~3-4 hours
**Prerequisites:** Azure tenant active, Entra ID P2 license confirmed, $50 budget alert configured

### Section 1: Create Identity Groups

We'll build role-based groups that map to real job functions in an AI-consuming organization.

**1.1 — Navigate to Groups**
1. Sign in to [entra.microsoft.com](https://entra.microsoft.com) with your admin account
2. Left sidebar → **Groups** → **All groups**
3. Click **+ New group**

**1.2 — Create Three Groups**

**Group A: AI-Admins**
- Group type: Security | Membership type: Assigned
- Description: `Full administrative access to AI workload — PIM required for activation`

**Group B: AI-Developers**
- Group type: Security | Membership type: Assigned
- Description: `Read/write access to AI resources — deploy, configure, and test`

**Group C: AI-Users**
- Group type: Security | Membership type: Assigned
- Description: `End-user access to consume AI services via approved interfaces only`

### Section 2: Create Test Users

**2.1 — Navigate to Users:** Left sidebar → **Users** → **All users** → **+ New user** → **Create new user**

**2.2 — User A: AI Admin**
- UPN: `sarah.admin@memanagementconsultingllc.onmicrosoft.com` | Display name: `Sarah Admin (Test)`
- Job title: `Cloud Security Engineer` | Department: `Security` | Group: `AI-Admins`

**2.3 — User B: AI Developer**
- UPN: `david.dev@memanagementconsultingllc.onmicrosoft.com` | Display name: `David Dev (Test)`
- Job title: `AI Engineer` | Department: `Engineering` | Group: `AI-Developers`

**2.4 — User C: AI End User**
- UPN: `emma.user@memanagementconsultingllc.onmicrosoft.com` | Display name: `Emma User (Test)`
- Job title: `Business Analyst` | Department: `Operations` | Group: `AI-Users`

### Section 3: Configure the Break-Glass Account

Break-glass accounts are emergency access accounts used only when all other authentication methods fail. They must be excluded from Conditional Access, protected with a strong offline-stored password, monitored for any usage, and never used for daily work.

**3.1 — Create the Account**
- UPN: `breakglass@memanagementconsultingllc.onmicrosoft.com` | Display name: `Break Glass Emergency Access`
- Password: 20+ characters, stored in a physical vault or offline password manager — not your regular manager
- Job title: `Emergency Access — Do Not Use`
- Assign role: **Global Administrator**

**3.2 — Document Break-Glass Handling**
In a real environment: password split between two people (dual custody), stored in a physical safe, monitored via a Sentinel alert on any sign-in, tested twice a year. Sentinel monitoring is configured in Phase 5.

### Section 4: Configure Named Locations

**4.1 — Trusted Location (United States)**
1. Entra portal → **Security** → **Protect** → **Conditional Access** → **Manage** → **Named locations**
2. **+ Countries location** → Name: `Trusted Countries — United States` → Countries: United States → Create

**4.2 — Blocked Location**
1. **+ Countries location** → Name: `Blocked Countries — High Risk`
2. Countries: North Korea, Iran, Russia, China, Belarus → Create

### Section 5: Build Conditional Access Policies

**5.1 — CA01: Require MFA for All Users**
- Users: Include All users, Exclude Break Glass Emergency Access
- Target resources: All cloud apps
- Grant: Require multifactor authentication
- State: **Report-only** initially (prevents lockout during setup, flip to On after testing)

**5.2 — CA02: Block Legacy Authentication**
- Users: Include All users, Exclude Break Glass Emergency Access
- Conditions: Client apps → uncheck Modern authentication clients, keep Exchange ActiveSync + Other clients
- Grant: Block access | State: **On**

**5.3 — CA03: Block Sign-Ins from Blocked Countries**
- Users: Include All users, Exclude Break Glass Emergency Access
- Conditions: Locations → Include `Blocked Countries — High Risk`
- Grant: Block access | State: **On**

**5.4 — CA04: Require Sign-In Every 4 Hours for AI Admins**
- Users: Include `AI-Admins`
- Session: Sign-in frequency → 4 hours
- State: **Report-only** initially

### Section 6: Configure Privileged Identity Management (PIM)

**6.1 — Enable PIM for AI-Admins**
1. Entra portal → **Identity governance** → **Privileged Identity Management** → **Microsoft Entra roles** → **Assignments** → **+ Add assignments**
2. Role: **Global Reader** (used instead of Global Admin to reduce risk while learning PIM)
3. Member: `Sarah Admin (Test)` → Assignment type: **Eligible** → Duration: Permanently eligible

**6.2 — Configure PIM Role Settings**
- Activation max duration: 4 hours
- Require: MFA on activation, justification, ticket information, approval to activate
- Approver: yourself
- Eligible assignments expire after 3 months
- Notifications on: assignment and activation

### Section 7: Configure Risk-Based Conditional Access Policies

Microsoft is retiring the standalone Identity Protection risk policies UI on **October 1, 2026**. Risk conditions are built directly into Conditional Access instead — a single policy surface with report-only testing support the legacy UI lacks. Don't combine user risk and sign-in risk conditions in the same policy; build them separately.

**7.1 — CA05: Require Risk Remediation for High User Risk**
- Users: Include All users, Exclude Break Glass Emergency Access
- Target resources: All resources
- Conditions: User risk → High
- Grant: Require risk remediation → Authentication strength: Multifactor authentication
- Session: Sign-in frequency → Every time (mandatory for risk policies)
- State: **Report-only** initially

**7.2 — CA06: Require MFA for Medium/High Sign-In Risk**
- Users: Include All users, Exclude Break Glass Emergency Access
- Conditions: Sign-in risk → High and Medium
- Grant: Authentication strength → Multifactor authentication
- Session: Sign-in frequency → Every time
- State: **Report-only** initially

**7.3 — Validate, Then Enforce**
Let both run in report-only, confirm expected matches in sign-in logs, then flip both to **On**.

### Section 8: Establish Governance Baseline with Azure Policy

Identity controls prevent unauthorized people from acting; Azure Policy prevents even authorized people from creating misconfigured resources.

**8.2 — Storage Accounts Should Restrict Network Access**
- Definitions → search policy → Assign → Scope: subscription → Effect: **Deny**

**8.3 — App Service Apps Should Use Managed Identity**
- Assign → Effect: **AuditIfNotExists** (start with audit; deny mode could break legitimate deployments while still learning)

**8.4 — Azure AI Services Resources Should Restrict Network Access**
- Assign → Effect: **Deny** (directly protects the future Azure OpenAI deployment)

**8.5 — Verify**
Policy → Assignments → confirm all three appear with correct scope. Wait 15-30 min for initial compliance scan, then check Compliance blade.

### Section 9: Testing & Validation

**9.1 — Test as Emma User**
Sign in as `emma.user@...` in a private browser window; MFA prompt should appear (or report-only log entry created).

**9.2 — Review Sign-In Logs**
> The Insights and reporting workbook requires a Log Analytics workspace (built in Phase 5) — it will 401 until then regardless of role. Use **Sign-in logs** instead.

Entra portal → **Monitoring** → **Sign-in logs** → find Emma's sign-in → open the **Conditional Access** tab to see policy results. Confirm **Authentication requirement: Multifactor authentication** and **Conditional Access: Success** (not `reportOnlySuccess`) to verify active enforcement, not just report-only logging.

**9.3 — Turn On Report-Only Policies**
Conditional Access → Policies → change CA01 and CA04 from Report-only to **On** → Save.

### Completion Checklist

- [ ] Three security groups created (AI-Admins, AI-Developers, AI-Users)
- [ ] Three test users created and assigned to groups
- [ ] Break-glass emergency access account configured with Global Admin role
- [ ] Break-glass password stored offline
- [ ] Two named locations created (Trusted US, Blocked High-Risk Countries)
- [ ] Four Conditional Access policies created and tested (CA01–CA04)
- [ ] PIM configured with just-in-time elevation for Sarah Admin
- [ ] CA05 and CA06 risk-based Conditional Access policies created and enforced
- [ ] Three Azure Policy baseline assignments configured
- [ ] Policy compliance scan completed
- [ ] Test sign-in as Emma User completed successfully
- [ ] All screenshots captured

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
