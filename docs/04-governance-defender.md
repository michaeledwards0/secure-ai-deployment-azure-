<div align="center">

# Phase 4: Governance & Defender for Cloud
### Extending the Governance Baseline and Enabling Workload Protection — Contoso AI Labs

</div>

---

## Overview

Phase 1 established a governance baseline of three Azure Policy assignments. Phase 4 extends that baseline across everything deployed since, and layers **Microsoft Defender for Cloud** on top for continuous workload protection — including **Defender for AI Services**, which specifically monitors for AI-workload threats like anomalous prompt patterns and unusual model access.

**Environment:** Personal Azure tenant (`contosoailabs.onmicrosoft.com`) | **Duration:** ~2 hours | **Standard:** SC-500 blueprint

### Design Rationale

**Why governance is revisited here rather than treated as "done" in Phase 1:** Policy assignments only govern resources that exist at assignment time or are created afterward within scope. Phase 1's policies were assigned before the network and AI resources existed — this phase confirms those policies actually caught anything, and adds new ones specific to the resources deployed in Phases 2 and 3.

**Why Defender for Cloud instead of relying on Azure Policy alone:** Azure Policy is preventive — it stops misconfigurations at creation time. Defender for Cloud is detective and adaptive — it continuously assesses posture, surfaces a Secure Score, and specifically watches for anomalous behavior on already-deployed resources that Policy alone can't catch after the fact.

**Why a Reader role for governance oversight:** Someone auditing compliance shouldn't need Contributor-level access to review Secure Score and compliance state — a read-only role scoped for oversight keeps the audit function separate from the ability to make changes.

---

## Case Study

### Objective
Extend governance coverage to every resource deployed in Phases 2 and 3, and enable continuous workload protection through Defender for Cloud — closing the gap between one-time policy assignment and ongoing security posture monitoring.

### Approach
*[Fill in after execution — describe what the initial compliance scan revealed, whether any resources were flagged non-compliant, and your reasoning for which Defender for Cloud plans were enabled given budget constraints.]*

### Controls Implemented
- Compliance verification of Phase 1's three baseline policies against Phase 2/3 resources
- Additional Azure Policy assignments covering network (deny public IP, enforced in Phase 2) and AI-specific misconfigurations
- Microsoft Defender for Cloud enabled, with **Defender for AI Services**, **Defender for Key Vault**, and **Defender for Servers** plans active
- Secure Score baseline captured
- Governance Reader role assigned for audit/oversight purposes, separate from administrative access

### RBAC Role Assignment

| Role | Scope | Rationale |
|---|---|---|
| `Reader` | `rg-secure-ai-prod` resource group | Allows compliance/audit review of Secure Score and policy state without granting any ability to modify resources |

### Frameworks Applied
- Microsoft Cloud Adoption Framework — Governance disciplines, Security Baseline
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 2 (Microsoft Defender for Cloud)
- NIST 800-53 — Continuous Monitoring control family

### Evidence
*[Screenshots added after execution — see capture list in the Execution Guide below.]*

### Lessons Learned
*[Fill in after execution.]*

---

## Next Phase

➡️ **[Phase 5: Detection Engineering](./05-detection-engineering.md)**

With governance and workload protection active, Phase 5 builds the Sentinel workspace and detection rules — including finally enabling the Insights and Reporting workbook flagged as pending back in Phase 1.

---

<details>
<summary><strong>📋 Full Execution Guide (click to expand)</strong> — step-by-step build instructions and completion checklist</summary>

<br>

**Prerequisites:** Phases 1–3 complete

### Section 1: Verify Existing Policy Compliance

1. Azure Portal → **Policy** → **Compliance**
2. Filter to `rg-secure-ai-prod`
3. Review compliance state for the three Phase 1 policies (deny public storage, audit managed identity, deny public AI services access) and the Phase 2 deny-public-IP policy
4. Document any non-compliant resources and remediate or note as accepted risk

📸 **Screenshot to capture:** Compliance dashboard showing current state across all assigned policies. Save as `screenshots/phase-04/01-policy-compliance-review.png`.

### Section 2: Enable Microsoft Defender for Cloud

1. Azure Portal → **Microsoft Defender for Cloud** → **Environment settings**
2. Select your subscription → **Defender plans**
3. Enable:
   - **Defender for AI Services** (protects Azure AI/OpenAI resources — anomalous prompt detection, model access monitoring)
   - **Defender for Key Vault**
   - **Defender for Servers** (Plan 1, sufficient for lab VMs)
4. Save

📸 **Screenshot to capture:** Defender plans page showing the three plans enabled. Save as `screenshots/phase-04/02-defender-plans-enabled.png`.

### Section 3: Capture Secure Score Baseline

1. Defender for Cloud → **Secure Score**
2. Review current score and the top recommendations
3. Address any quick wins that don't require resources from later phases (e.g., enabling diagnostic settings, which also prepares for Phase 5)

📸 **Screenshot to capture:** Secure Score overview page. Save as `screenshots/phase-04/03-secure-score-baseline.png`.

### Section 4: Assign Governance Reader Role

1. `rg-secure-ai-prod` → **Access control (IAM)** → **+ Add role assignment**
2. Role: **Reader** → assign to yourself or a dedicated audit identity if you create one
3. Save

📸 **Screenshot to capture:** Role assignments showing the Reader role added. Save as `screenshots/phase-04/04-reader-role-assigned.png`.

### Section 5: Assign Additional Policy Coverage

1. Policy → **Definitions** → search for a built-in initiative covering AI/Cognitive Services security recommendations (or use the custom definitions in `policies/require-diagnostic-settings.json` from this repo)
2. Assign at the `rg-secure-ai-prod` scope
3. Verify assignment

📸 **Screenshot to capture:** Policy assignments list showing the full governance baseline (Phase 1 + Phase 2 + Phase 4 policies). Save as `screenshots/phase-04/05-full-governance-baseline.png`.

### Completion Checklist

- [ ] Phase 1 policy compliance verified against Phase 2/3 resources
- [ ] Defender for AI Services, Defender for Key Vault, and Defender for Servers enabled
- [ ] Secure Score baseline captured
- [ ] Reader role assigned for governance oversight
- [ ] Additional diagnostic-settings/AI-specific policy assigned
- [ ] All screenshots captured and saved to `screenshots/phase-04/`

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
