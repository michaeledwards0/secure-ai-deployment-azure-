# Phase 1: Identity Fortress

## Building the Zero Trust Identity Layer for AI Workload Access

**Duration:** ~3-4 hours  
**Prerequisites:** Azure tenant active, Entra ID P2 license confirmed, $50 budget alert configured  
**Focus Areas:** Identity, Access Management, Zero Trust, Foundational Governance

---

## Why This Phase Comes First

In Zero Trust architecture, identity is the perimeter. Before any resources exist, before the AI service is deployed, before a single line of code runs — the identity layer must be established with least-privilege access, strong authentication, and just-in-time elevation.

This phase demonstrates the identity architecture that would protect a real production AI workload from the two most common attack paths: **compromised credentials** and **privilege escalation**. It also establishes the first governance guardrails via Azure Policy — because policy that isn't enforced isn't policy.

**By the end of this phase, you will have:**
- A production-grade identity structure with role-based groups
- A properly configured break-glass emergency access account
- Conditional Access policies enforcing MFA, geographic restrictions, and legacy authentication blocks
- Privileged Identity Management (PIM) configured for just-in-time admin elevation
- Baseline Azure Policy assignments enforcing governance guardrails
- All controls documented with screenshots for the case study

---

## Section 1: Create Identity Groups

We'll build role-based groups that map to real job functions in an AI-consuming organization.

### 1.1 — Navigate to Groups

1. Sign in to **[entra.microsoft.com](https://entra.microsoft.com)** with your admin account 
2. Left sidebar → **Groups** → **All groups**
3. Click **+ New group**

### 1.2 — Create Three Groups

Create each of the following, one at a time. Use these exact settings for each:

**Group A: AI-Admins**
- **Group type:** Security
- **Group name:** `AI-Admins`
- **Group description:** `Full administrative access to AI workload — PIM required for activation`
- **Membership type:** Assigned
- **Owners:** (leave empty for now)
- **Members:** (leave empty — we'll assign users in Section 2)
- Click **Create**

**Group B: AI-Developers**
- **Group type:** Security
- **Group name:** `AI-Developers`
- **Group description:** `Read/write access to AI resources — deploy, configure, and test`
- **Membership type:** Assigned
- Click **Create**

**Group C: AI-Users**
- **Group type:** Security
- **Group name:** `AI-Users`
- **Group description:** `End-user access to consume AI services via approved interfaces only`
- **Membership type:** Assigned
- Click **Create**

📸 **Screenshot to capture:** The Groups page showing all three groups created. Save as `screenshots/phase-01/01-groups-created.png`.

---

## Section 2: Create Test Users

Create three test users mapping to each group. These simulate the identity types in a real organization.

### 2.1 — Navigate to Users

1. Left sidebar → **Users** → **All users**
2. Click **+ New user** → **Create new user**

### 2.2 — Create User A: AI Admin

- **User principal name:** `sarah.admin@memanagementconsultingllc.onmicrosoft.com`
- **Display name:** `Sarah Admin (Test)`
- **Password:** Auto-generate (save it in Key Vault later)
- **Usage location:** United States
- **Job title:** `Cloud Security Engineer`
- **Department:** `Security`
- Click **Next: Properties** → skip → **Next: Assignments**
- **Add group:** `AI-Admins`
- Click **Review + create** → **Create**

### 2.3 — Create User B: AI Developer

- **User principal name:** `david.dev@memanagementconsultingllc.onmicrosoft.com`
- **Display name:** `David Dev (Test)`
- **Password:** Auto-generate
- **Usage location:** United States
- **Job title:** `AI Engineer`
- **Department:** `Engineering`
- **Add group:** `AI-Developers`
- Click **Create**

### 2.4 — Create User C: AI End User

- **User principal name:** `emma.user@memanagementconsultingllc.onmicrosoft.com`
- **Display name:** `Emma User (Test)`
- **Password:** Auto-generate
- **Usage location:** United States
- **Job title:** `Business Analyst`
- **Department:** `Operations`
- **Add group:** `AI-Users`
- Click **Create**

📸 **Screenshot to capture:** The Users page showing all three test users. Save as `screenshots/phase-01/02-users-created.png`.

---

## Section 3: Configure the Break-Glass Account

Break-glass accounts are emergency access accounts used only when all other authentication methods fail. They must be:

- **Excluded** from Conditional Access policies (so they always work in an emergency)
- **Protected** with a very strong, offline-stored password
- **Monitored** for any usage (sign-in should be a serious alert)
- **Never** used for daily work

### 3.1 — Create the Break-Glass Account

1. Users → **+ New user** → **Create new user**
2. Fill in:
   - **User principal name:** `breakglass@memanagementconsultingllc.onmicrosoft.com`
   - **Display name:** `Break Glass Emergency Access`
   - **Password:** Generate a strong 20+ character password. **Store this in a physical vault or offline password manager — NOT in your regular password manager.**
   - **Usage location:** United States
   - **Job title:** `Emergency Access — Do Not Use`
3. Click **Next: Assignments**
4. **Assign role:** **Global Administrator**
5. Click **Review + create** → **Create**

### 3.2 — Document Break-Glass Handling

In a real environment, this account would be:
- Password split between two people (dual custody)
- Stored in a physical safe
- Monitored via a Sentinel alert on any sign-in
- Tested twice a year

We'll set up the Sentinel monitoring alert in Phase 5.

📸 **Screenshot to capture:** The break-glass user's details page showing Global Administrator role assigned. Save as `screenshots/phase-01/03-break-glass-created.png`.

---

## Section 4: Configure Named Locations

Named locations let Conditional Access make decisions based on where a sign-in originates. We'll define our "trusted" and "untrusted" regions.

### 4.1 — Create Trusted Location (United States)

1. Entra portal → **Security** (left sidebar) → **Protect** → **Conditional Access** → **Manage** → **Named locations**
2. Click **+ Countries location**
3. **Name:** `Trusted Countries — United States`
4. **Countries/Regions:** Select **United States**
5. Leave **"Determine location by IP address"** selected
6. Click **Create**

### 4.2 — Create Blocked Location

1. Click **+ Countries location**
2. **Name:** `Blocked Countries — High Risk`
3. **Countries/Regions:** Select these known high-risk regions:
   - North Korea
   - Iran
   - Russia
   - China
   - Belarus
4. Click **Create**

📸 **Screenshot to capture:** Named locations page showing both locations. Save as `screenshots/phase-01/04-named-locations.png`.

---

## Section 5: Build Conditional Access Policies

This is the core of Zero Trust identity. We'll build four foundational policies.

### 5.1 — Policy 1: Require MFA for All Users

1. Entra portal → **Protection** → **Conditional Access** → **Policies**
2. Click **+ New policy**
3. **Name:** `CA01 — Require MFA for All Users`
4. **Users:**
   - Include: **All users**
   - Exclude: **Users and groups** → select `Break Glass Emergency Access`
5. **Target resources:**
   - Cloud apps: **All cloud apps**
6. **Conditions:** (leave default)
7. **Grant:**
   - Select **Grant access**
   - Check **Require multifactor authentication**
   - Click **Select**
8. **Enable policy:** **Report-only** for now (we'll flip to On after testing)
9. Click **Create**

> ⚠️ **Why Report-only first:** Report-only mode logs what *would* have happened without actually enforcing. This prevents you from locking yourself out during setup.

### 5.2 — Policy 2: Block Legacy Authentication

Legacy auth protocols (POP, IMAP, older Exchange) don't support MFA and are the #1 vector for password spray attacks.

1. Click **+ New policy**
2. **Name:** `CA02 — Block Legacy Authentication`
3. **Users:** Include **All users**, Exclude **Break Glass Emergency Access**
4. **Target resources:** All cloud apps
5. **Conditions:**
   - **Client apps** → **Configure: Yes**
   - Uncheck **Modern authentication clients**
   - Keep checked: **Exchange ActiveSync clients** and **Other clients**
   - Click **Done**
6. **Grant:** Select **Block access** → **Select**
7. **Enable policy:** **On** (this one is safe to enforce immediately)
8. Click **Create**

### 5.3 — Policy 3: Block Sign-Ins from Blocked Countries

1. Click **+ New policy**
2. **Name:** `CA03 — Block Sign-Ins from High-Risk Countries`
3. **Users:** Include **All users**, Exclude **Break Glass Emergency Access**
4. **Target resources:** All cloud apps
5. **Conditions:**
   - **Locations** → **Configure: Yes**
   - **Include:** **Selected locations** → `Blocked Countries — High Risk`
6. **Grant:** Select **Block access** → **Select**
7. **Enable policy:** **On**
8. Click **Create**

### 5.4 — Policy 4: Require Sign-In Frequency for AI Admins

Admin accounts should re-authenticate more often than regular users.

1. Click **+ New policy**
2. **Name:** `CA04 — Require Sign-In Every 4 Hours for AI Admins`
3. **Users:** Include **Users and groups** → `AI-Admins`
4. **Target resources:** All cloud apps
5. **Session:**
   - Check **Sign-in frequency**
   - Set to **4** hours
   - Click **Select**
6. **Enable policy:** **Report-only**
7. Click **Create**

📸 **Screenshot to capture:** Conditional Access Policies page showing all four policies. Save as `screenshots/phase-01/05-conditional-access-policies.png`.

---

## Section 6: Configure Privileged Identity Management (PIM)

PIM enables just-in-time (JIT) elevation for admin roles. Instead of users being permanent admins, they request elevation for a specific window with approval.

### 6.1 — Enable PIM for the AI-Admins Group

1. Entra portal → **Identity governance** → **Privileged Identity Management**
2. Left sidebar → **Microsoft Entra roles**
3. Click **Assignments** → **+ Add assignments**
4. **Select role:** **Global Reader** (for testing; we're using this instead of Global Admin to reduce risk while learning PIM)
5. **Select members:** Add `Sarah Admin (Test)`
6. Click **Next**
7. **Assignment type:** **Eligible** (this means she can activate the role JIT, not that she has it permanently)
8. **Duration:** Permanently eligible
9. Click **Assign**

### 6.2 — Configure PIM Role Settings

1. Still in PIM → **Microsoft Entra roles** → **Settings**
2. Click **Global Reader**
3. Click **Edit**

**Activation tab:**
- **Activation maximum duration (hours):** 4
- **On activation, require:** Microsoft Entra Multifactor Authentication
- **Require justification on activation:** ✓
- **Require ticket information on activation:** ✓
- **Require approval to activate:** ✓
- **Approvers:** Add yourself (`michaeledwards@...`)

**Assignment tab:**
- **Allow permanent eligible assignment:** Off
- **Expire eligible assignments after:** 3 months

**Notification tab:**
- **Send notifications when members are assigned as eligible:** ✓
- **Send notifications when eligible members activate:** ✓

Click **Update**

📸 **Screenshot to capture:** PIM Assignments page showing Sarah as eligible for Global Reader. Save as `screenshots/phase-01/06-pim-configured.png`.

---

## Section 7: Configure Risk-Based Conditional Access Policies

Microsoft has deprecated the standalone Identity Protection risk policies UI — the legacy user risk and sign-in risk policies configured directly in ID Protection are retiring on **October 1, 2026**. Current Microsoft guidance is to build risk conditions directly into Conditional Access instead. This gives a single policy management surface, report-only testing before enforcement, and better sign-in log diagnostics showing exactly which risk policy applied to a given sign-in.

> **Note:** Don't combine user risk and sign-in risk conditions in the same policy — Microsoft recommends creating separate policies for each, as below.

### 7.1 — CA05: Require Risk Remediation for High User Risk

1. Entra portal → **Security** → **Protect** → **Conditional Access** → **Policies** → **+ New policy**
2. **Name:** `CA05 — Require Remediation for High User Risk`
3. **Assignments → Users:**
   - Include: **All users**
   - Exclude: **Users and groups** → `Break Glass Emergency Access`
4. **Target resources → Include:** **All resources** (formerly "All cloud apps")
5. **Conditions → User risk:**
   - Configure: **Yes**
   - Risk level: **High**
6. **Grant:**
   - Select **Grant access**
   - Check **Require risk remediation** (this auto-selects **Require authentication strength**)
   - Authentication strength: **Multifactor authentication**
7. **Session:** Sign-in frequency → **Every time** (auto-applied and mandatory for risk-based policies)
8. **Enable policy:** **Report-only** to start
9. Click **Create**

### 7.2 — CA06: Require MFA for Medium/High Sign-In Risk

1. Click **+ New policy**
2. **Name:** `CA06 — Require MFA for Medium and High Sign-In Risk`
3. **Assignments → Users:**
   - Include: **All users**
   - Exclude: `Break Glass Emergency Access`
4. **Target resources → Include:** **All resources**
5. **Conditions → Sign-in risk:**
   - Configure: **Yes**
   - Risk level: **High and Medium**
6. **Grant:**
   - Select **Grant access**
   - **Require authentication strength** → **Multifactor authentication**
7. **Session:** Sign-in frequency → **Every time**
8. **Enable policy:** **Report-only** to start
9. Click **Create**

### 7.3 — Validate, Then Enforce

1. Let both policies run in report-only for a few sign-ins (or trigger a test via the Insights and Reporting workbook)
2. Entra portal → **Conditional Access** → **Insights and reporting** → confirm CA05 and CA06 show expected report-only matches
3. Flip both **CA05** and **CA06** from **Report-only** to **On**

📸 **Screenshot to capture:** Conditional Access Policies page showing CA05 and CA06 alongside CA01–CA04. Save as `screenshots/phase-01/07-risk-based-ca-policies.png`.

---

## Section 8: Establish Governance Baseline with Azure Policy

Identity controls prevent unauthorized *people* from acting. Azure Policy prevents even authorized people from creating *misconfigured resources*. We'll assign three foundational policies now — they'll be enforced across every phase that follows.

### 8.1 — Why This Matters

A common attack pattern is not "hacker breaks in" — it's "authorized admin makes a mistake and exposes a resource to the public internet." Azure Policy is the guardrail that stops those mistakes before they become breaches. In real cloud security roles, policy authoring is a core responsibility.

### 8.2 — Assign Built-In Policy: Storage Accounts Should Restrict Network Access

Prevents any storage account (which our AI workload will use) from being publicly exposed.

1. Azure Portal → search **"Policy"** → click it
2. Left sidebar → **Authoring** → **Definitions**
3. Search: `Storage accounts should restrict network access`
4. Click the policy definition
5. Click **Assign**
6. **Scope:** Your subscription
7. **Basics:**
   - **Assignment name:** `Deny Public Storage Accounts`
   - **Policy enforcement:** **Enabled**
8. **Parameters:**
   - **Effect:** `Deny`
9. Click **Review + create** → **Create**

### 8.3 — Assign Built-In Policy: App Service apps should use managed identity

Enforces that resources use managed identities instead of stored credentials.

1. Definitions → search: `App Service apps should use managed identity`
2. Click the policy → **Assign**
3. **Scope:** Your subscription
4. **Assignment name:** `Require Managed Identity`
5. **Parameters → Effect:** `AuditIfNotExists` (start with audit to see impact)
6. Click **Review + create** → **Create**

> **Why start with Audit:** Deny mode could break legitimate deployments while you're still learning. Audit surfaces violations without blocking, then you can flip to Deny once confident.

### 8.4 — Assign Built-In Policy: Azure AI Services resources should restrict network access

Directly protects your future Azure OpenAI deployment.

1. Definitions → search: `Azure AI Services resources should restrict network access`
2. Click the policy → **Assign**
3. **Scope:** Your subscription
4. **Assignment name:** `Deny Public Access on Azure AI Services`
5. **Parameters → Effect:** `Deny`
6. Click **Review + create** → **Create**

### 8.5 — Verify Policy Assignments

1. Policy → **Assignments**
2. Confirm all three policies appear with correct scope
3. Wait 15-30 minutes for initial compliance scan to complete
4. Navigate to **Compliance** → your subscription should show initial compliance state

📸 **Screenshot to capture:** Policy Assignments page showing all three assigned policies. Save as `screenshots/phase-01/08-azure-policy-baseline.png`.

---

## Section 9: Testing & Validation

### 9.1 — Test as Emma User

1. Open a private browser window
2. Go to **[portal.azure.com](https://portal.azure.com)**
3. Sign in as `emma.user@memanagementconsultingllc.onmicrosoft.com`
4. **Expected behavior:** MFA prompt should appear (or a report-only log entry created)
5. Complete MFA setup with your phone
6. Sign in successfully

### 9.2 — Review Sign-In Logs

> **Note:** The **Insights and reporting** workbook (Conditional Access → Insights and reporting) requires a Log Analytics workspace with Entra diagnostic settings streaming into it — infrastructure that doesn't exist until Phase 5. Visiting it now returns a 401 "You don't have access" error regardless of role; this isn't a permissions problem, it's a missing data pipe. Use **Sign-in logs** instead — it reads the same underlying data with no workspace dependency.

1. Back in your admin session
2. Entra portal → **Monitoring** → **Sign-in logs**
3. Find Emma's sign-in (filter by user if needed)
4. Click into the sign-in → open the **Conditional Access** tab on the detail pane to see every policy evaluated and its result (Success, Failure, Not applied, or Report-only would-have-applied)
5. Also check the **Authentication requirement** and **Conditional Access** columns in the grid view (or export to CSV):
   - **Authentication requirement: Multifactor authentication** confirms MFA was required for that sign-in
   - **Conditional Access: Success** (not `reportOnlySuccess`) confirms the policy is actively enforcing, not just logging in report-only mode
6. This confirms CA01 (and any other enforced policy) is working correctly

> **Once Phase 5's Log Analytics workspace exists:** add a diagnostic setting on Microsoft Entra ID sending `SignInLogs` and `AuditLogs` to that workspace, and the Insights and reporting workbook comes online — giving trend charts and per-policy impact summaries on top of the same raw data.


### 9.3 — Turn On Report-Only Policies

Now that testing confirms the policies work:

1. Go back to **Conditional Access** → **Policies**
2. Click **CA01 — Require MFA for All Users**
3. Change **Enable policy** from **Report-only** to **On**
4. Save
5. Do the same for **CA04 — Require Sign-In Every 4 Hours for AI Admins**

📸 **Screenshot to capture:** Insights and reporting dashboard showing policy application. Save as `screenshots/phase-01/09-testing-validation.png`.

---

## Phase 1 Completion Checklist

- [ ] Three security groups created (AI-Admins, AI-Developers, AI-Users)
- [ ] Three test users created and assigned to groups
- [ ] Break-glass emergency access account configured with Global Admin role
- [ ] Break-glass password stored offline (physical vault or paper copy)
- [ ] Two named locations created (Trusted US, Blocked High-Risk Countries)
- [ ] Four Conditional Access policies created and tested (CA01–CA04)
- [ ] PIM configured with just-in-time elevation for Sarah Admin
- [ ] CA05 and CA06 risk-based Conditional Access policies created and enforced
- [ ] Three Azure Policy baseline assignments configured
- [ ] Policy compliance scan completed (initial state captured)
- [ ] Test sign-in as Emma User completed successfully
- [ ] All screenshots captured and saved to `screenshots/phase-01/`

---

## Case Study Writeup — Phase 1

*This section belongs at the end of the Phase 1 doc after execution. Write it once complete.*

### Objective
Establish the Zero Trust identity layer and governance baseline for the Contoso AI Labs environment before any AI workloads are deployed, following the principle that identity is the primary security perimeter in cloud architectures — and that policy enforcement is what keeps that perimeter intact over time.

### Approach
[Describe your reasoning for the group structure, why break-glass matters, why report-only mode was used first, how PIM enables least-privilege operations, why risk-based Conditional Access was built directly rather than through the legacy Identity Protection UI, and why policy-as-guardrails matters at the identity layer.]

### Controls Implemented
- Role-based access control via three security groups
- MFA enforcement for all users except break-glass
- Legacy authentication blocked (eliminates ~30% of common attack surface)
- Geographic access restrictions blocking known high-risk regions
- Just-in-time privileged access via PIM with approval workflow
- Risk-based Conditional Access policies (CA05, CA06) built on ID Protection signals, ahead of the legacy risk policies' October 2026 retirement
- Azure Policy guardrails preventing common misconfigurations at creation time

### Frameworks Applied
- NIST 800-207 (Zero Trust Architecture)
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 1 (Identity and Access Management)
- Microsoft Cloud Adoption Framework — Security Baseline for Identity
- Microsoft Cloud Adoption Framework — Governance disciplines

### Lessons Learned
[Fill in after execution — what surprised you, what took longer than expected, what would you do differently in a real environment.]

---

## Next Phase

➡️ **[Phase 2: Network Architecture & Isolation](./02-network-architecture.md)**

In Phase 2 we'll build the network layer that will house the AI workload — hub-spoke topology, private endpoints, NSGs, and Azure Bastion for admin access. This is where the workload starts to take shape.

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
