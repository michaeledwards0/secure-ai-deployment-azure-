<div align="center">

# Phase 1 Runbook: Identity Fortress
### Step-by-Step Implementation Guide for the Contoso AI Labs Identity and Governance Baseline

</div>

---

## Purpose

This runbook documents the implementation steps used to establish Phase 1 identity, privileged-access, Conditional Access, emergency-access, and Azure governance controls.

Use the accompanying [Phase 1 portfolio case study](../01-identity-fortress.md) for the executive summary, architecture, engineering decisions, results, and lessons learned.

---

## Scope

- Security groups and test identities
- Emergency-access account
- Named locations
- Conditional Access policies CA01–CA06
- Privileged Identity Management
- Azure Policy assignments
- Management Group creation and policy inheritance
- Testing and validation

## Prerequisites

- Active Azure tenant
- Microsoft Entra ID P2 licensing
- Administrative access to Microsoft Entra ID
- Azure subscription
- Budget alert configured
- Private browser session for test-user validation

**Estimated time:** 3–4 hours

---

# 1. Create Identity Groups

Create function-based security groups before provisioning users.

## 1.1 Create `AI-Admins`

- Group type: Security
- Membership type: Assigned
- Description: `Full administrative access to AI workload — PIM required for activation`

## 1.2 Create `AI-Developers`

- Group type: Security
- Membership type: Assigned
- Description: `Read/write access to AI resources — deploy, configure, and test`

## 1.3 Create `AI-Users`

- Group type: Security
- Membership type: Assigned
- Description: `End-user access to consume AI services through approved interfaces only`

**Validation:** Confirm all three groups appear under **Groups → All groups**.

---

# 2. Create Test Users

| User | Job function | Group |
|---|---|---|
| `sarah.admin@<tenant>.onmicrosoft.com` | Cloud Security Engineer | `AI-Admins` |
| `david.dev@<tenant>.onmicrosoft.com` | AI Engineer | `AI-Developers` |
| `emma.user@<tenant>.onmicrosoft.com` | Business Analyst | `AI-Users` |

For each identity:

1. Create the user in Microsoft Entra ID.
2. Assign the correct job title and department.
3. Add the user to the intended group.
4. Confirm the user is not a member of unintended privileged groups.

---

# 3. Configure the Break-Glass Account

## 3.1 Create the Account

- UPN: `breakglass@<tenant>.onmicrosoft.com`
- Display name: `Break Glass Emergency Access`
- Job title: `Emergency Access — Do Not Use`
- Role: Global Administrator

## 3.2 Protect the Credential

- Use a unique password of at least 20 characters.
- Store it offline or in a secured emergency-access process.
- Never use it for routine administration.
- Exclude it from all Conditional Access policies.

## 3.3 Document Operations

In an enterprise environment, use dual custody, periodically test the account, alert on all sign-ins, and include it in access reviews. Sentinel monitoring is implemented in Phase 5.

---

# 4. Configure Named Locations

## 4.1 Trusted Location

Create `Trusted Countries — United States` and select the United States.

## 4.2 Blocked Location

Create `Blocked Countries — High Risk` and select North Korea, Iran, Russia, China, and Belarus.

**Validation:** Confirm both named locations appear in Conditional Access.

---

# 5. Build Core Conditional Access Policies

> Exclude the break-glass account from every Conditional Access policy.

## CA01 — Require MFA for All Users

- Include: All users
- Exclude: Break-glass account
- Target: All cloud apps/resources
- Grant: Require MFA
- Initial state: Report-only

## CA02 — Block Legacy Authentication

- Include: All users
- Exclude: Break-glass account
- Client apps: Legacy authentication clients
- Grant: Block access
- State: On

## CA03 — Block High-Risk Countries

- Include: All users
- Exclude: Break-glass account
- Location: `Blocked Countries — High Risk`
- Grant: Block access
- State: On

## CA04 — Admin Sign-In Frequency

- Include: `AI-Admins`
- Session control: Four-hour sign-in frequency
- Initial state: Report-only

---

# 6. Configure Privileged Identity Management

1. Open **Identity governance → Privileged Identity Management**.
2. Select **Microsoft Entra roles → Assignments → Add assignments**.
3. Configure:
   - Role: Global Reader
   - Member: Sarah Admin
   - Assignment type: Eligible
4. Configure activation controls:
   - Maximum duration: Four hours
   - Require MFA
   - Require approval
   - Require justification
   - Require ticket information
   - Enable assignment and activation notifications

**Validation:** Sarah Admin should appear under eligible assignments, not active assignments.

---

# 7. Configure Risk-Based Conditional Access

## CA05 — High User Risk Remediation

- Include: All users
- Exclude: Break-glass account
- Target: All resources
- User risk: High
- Grant: Require risk remediation and MFA authentication strength
- Session: Sign-in frequency every time
- Initial state: Report-only

## CA06 — Medium/High Sign-In Risk

- Include: All users
- Exclude: Break-glass account
- Sign-in risk: Medium and High
- Grant: Require MFA authentication strength
- Session: Sign-in frequency every time
- Initial state: Report-only

Review report-only results before enabling either policy.

---

# 8. Establish the Azure Policy Baseline

## 8.1 Require Managed Identity

1. Open **Azure Policy → Definitions**.
2. Search for `App Service apps should use managed identity`.
3. Assign at subscription scope.
4. Name the assignment `Require Managed Identity`.
5. Set effect to `AuditIfNotExists`.

## 8.2 Restrict Azure AI Services Network Access

1. Search for `Azure AI Services resources should restrict network access`.
2. Assign at subscription scope.
3. Name the assignment `Deny Public Access on Azure AI Services`.
4. Set effect to Deny.

## 8.3 Validate

Confirm both assignments and allow time for compliance evaluation.

---

# 9. Create the Management Group and Demonstrate Inheritance

## 9.1 Create the Management Group

- ID: `mg-contoso-ai-labs`
- Display name: `Contoso AI Labs`

## 9.2 Move the Subscription

Place the lab subscription beneath the new Management Group.

## 9.3 Assign Storage Network Policy

1. Search for `Storage accounts should restrict network access`.
2. Assign it at `mg-contoso-ai-labs` scope.
3. Name it `Deny Public Storage Accounts`.
4. Set effect to Deny.

## 9.4 Validate Inheritance

Open the subscription compliance view and confirm the policy appears without a duplicate subscription-level assignment.

---

# 10. Test and Validate

## 10.1 Test as Emma User

1. Open a private browser session.
2. Sign in as Emma User.
3. Complete MFA as prompted.

## 10.2 Review Sign-In Logs

1. Open **Monitoring → Sign-in logs**.
2. Select the Emma User sign-in.
3. Review the **Conditional Access** tab.
4. Confirm MFA was required and the expected policy succeeded.

The Insights and Reporting workbook depends on Log Analytics integration added in Phase 5, so use sign-in logs for Phase 1 validation.

## 10.3 Enable Validated Policies

After confirming report-only behavior, enable CA01, CA04, CA05, and CA06 as appropriate.

---

# 11. Completion Checklist

## Identity

- [ ] Three security groups created
- [ ] Three test users created
- [ ] Users assigned to correct groups

## Emergency Access

- [ ] Break-glass account created
- [ ] Global Administrator assigned
- [ ] Password stored offline
- [ ] Account excluded from all Conditional Access policies
- [ ] Monitoring requirement documented

## Conditional Access

- [ ] Named locations created
- [ ] CA01–CA06 created
- [ ] Report-only results reviewed
- [ ] Validated policies enabled

## Privileged Access

- [ ] Sarah Admin configured as PIM eligible
- [ ] MFA, approval, justification, and ticket information required
- [ ] Activation duration limited
- [ ] Notifications enabled

## Governance

- [ ] Managed identity policy assigned
- [ ] Azure AI network policy assigned
- [ ] Management Group created
- [ ] Subscription placed beneath Management Group
- [ ] Storage policy assigned at Management Group scope
- [ ] Inheritance confirmed
- [ ] Compliance evaluation reviewed

## Evidence

- [ ] Groups
- [ ] Users
- [ ] Break-glass account
- [ ] Named locations
- [ ] Core Conditional Access policies
- [ ] PIM assignment
- [ ] Risk policies
- [ ] Policy baseline
- [ ] Management Group
- [ ] Policy inheritance
- [ ] Enforced sign-in result

---

# 12. Troubleshooting

## Conditional Access does not apply

- Confirm the user is included and not excluded.
- Confirm the target resource is in scope.
- Check whether the policy is report-only.
- Review the sign-in log's Conditional Access tab.

## Risk policies do not trigger

- Confirm Entra ID licensing.
- Confirm the configured risk level.
- Normal sign-ins may not generate risky events.

## Policy compliance is delayed

- Allow time for evaluation.
- Confirm assignment scope.
- Review exemptions and exclusions.
- Trigger a compliance scan where available.

## PIM activation is unavailable

- Confirm the assignment is eligible.
- Confirm it has not expired.
- Verify activation settings and licensing.

## Break-glass account is blocked

- Review exclusions immediately.
- Maintain another functioning administrative session while making changes.

---

## Related Documentation

- [Phase 1 Portfolio Case Study](../01-identity-fortress.md)
- [Phase 2 — Network Architecture & Isolation](../02-network-architecture.md)
- [Project Overview](../../README.md)

---

<div align="center">

**Runbook complete — preserve a tested emergency-access path throughout implementation.**

</div>
