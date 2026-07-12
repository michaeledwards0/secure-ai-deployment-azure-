<div align="center">

# Phase 3: Azure AI Services (OpenAI) Deployment
### Deploying the AI Workload — Contoso AI Labs

</div>

---

## Overview

With identity (Phase 1) and network isolation (Phase 2) in place, Phase 3 deploys the actual AI workload: an **Azure AI services** resource with the Azure OpenAI model deployment, connected exclusively through the private endpoint subnet built in Phase 2. This is also the phase where the Phase 1 identity groups stop being inert containers and become functional — Azure RBAC role assignments are applied here for the first time, connecting AI-Admins, AI-Developers, and AI-Users to real permissions on a real resource.

**Environment:** Personal Azure tenant (`contosoailabs.onmicrosoft.com`) | **Duration:** ~2-3 hours | **Standard:** SC-500 blueprint

> **Naming note:** Microsoft has rebranded the product family from "Cognitive Services" to **Azure AI services.** However, the built-in RBAC role names have not been renamed to match — roles like `Cognitive Services OpenAI Contributor` and `Cognitive Services OpenAI User` still carry the legacy name in the role catalog itself. This doc refers to the resource as Azure AI services throughout, but role assignment screenshots will show the legacy role names, since that's what Azure currently exposes.

### Design Rationale

**Why RBAC belongs in this phase, not Phase 1:** Azure RBAC assigns permissions to *resources*. In Phase 1, the AI-Admins/Developers/Users groups were created, but no resource existed yet to scope a role assignment to — assigning roles at that point would have had nothing to attach to. Phase 3 is the first point where the group structure becomes enforceable rather than organizational.

**Why managed identity instead of API keys:** A system-assigned managed identity lets the AI resource authenticate to Key Vault (for customer-managed keys) and other Azure services without any credential ever being stored in code or configuration. This eliminates an entire class of credential-leak risk.

**Why customer-managed keys (CMK):** Default encryption at rest uses Microsoft-managed keys. CMK moves key ownership into your own Key Vault, meaning you control key rotation and revocation independently of Microsoft — a common compliance requirement in regulated industries.

**Why content filters at deployment, not as an afterthought:** Azure OpenAI's content filtering is configured per-deployment. Setting it at creation, rather than retrofitting later, means no model deployment in this environment is ever live without a filtering policy attached.

---

## Case Study

### Objective
Deploy the Azure AI services (Azure OpenAI) workload into the isolated network from Phase 2, with no public network exposure, encryption keys under organizational control, and access permissions enforced through the identity groups established in Phase 1 — closing the loop between identity design and actual resource authorization.

### Approach
*[Fill in after execution — describe your reasoning for the CMK key rotation policy, why specific content filter severities were chosen, and any tradeoffs between model deployment capacity and budget constraints.]*

### Controls Implemented
- Azure AI services resource deployed with public network access disabled, reachable only via the Phase 2 private endpoint
- System-assigned managed identity for Key Vault authentication
- Customer-managed keys for encryption at rest, stored in a dedicated Key Vault with access policies scoped to the managed identity only
- Content filtering configured at the model deployment level
- Azure RBAC role assignments connecting Phase 1's identity groups to real permissions on the AI resource
- Private DNS zone group linking the private endpoint to the zone created in Phase 2

### RBAC Role Assignments

| Group | Role | Scope | Rationale |
|---|---|---|---|
| **AI-Admins** | `Contributor` | `rg-secure-ai-prod` resource group | Full administrative control to manage all resources in the environment, consistent with the group's Phase 1 description |
| **AI-Developers** | `Cognitive Services OpenAI Contributor` | Azure AI services resource | Deploy models, fine-tune, and generate content — matches the "deploy, configure, and test" scope defined in Phase 1 |
| **AI-Users** | `Cognitive Services OpenAI User` | Azure AI services resource | Inference and content generation only, no deployment or configuration rights — matches "consume AI services via approved interfaces only" |

### Frameworks Applied
- Microsoft Cloud Adoption Framework — Data Encryption and Key Management
- OWASP Top 10 for LLM Applications — LLM06 (Sensitive Information Disclosure), addressed via network isolation and CMK
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 4 (Storage/Key Management analog applied to Cognitive/AI resources)
- NIST 800-207 (Zero Trust Architecture) — least-privilege authorization

### Evidence
*[Screenshots added after execution — see capture list in the Execution Guide below.]*

### Lessons Learned
*[Fill in after execution.]*

---

## Next Phase

➡️ **[Phase 4: Governance & Defender for Cloud](./04-governance-defender.md)**

With the workload deployed, Phase 4 layers Defender for Cloud workload protection and an expanded governance baseline over everything built so far.

---

<details>
<summary><strong>📋 Full Execution Guide (click to expand)</strong> — step-by-step build instructions and completion checklist</summary>

<br>

**Prerequisites:** Phase 1 and Phase 2 complete, `snet-private-endpoints` and `privatelink.openai.azure.com` DNS zone already exist

### Section 1: Deploy the Key Vault

1. Azure Portal → **Key Vaults** → **+ Create**
   - Resource group: `rg-secure-ai-prod`
   - Name: `kv-contoso-ai` (must be globally unique — append a suffix if needed)
   - Region: consistent with prior phases
   - Pricing tier: Standard
2. **Networking tab:** Disable public access → connect via **Private endpoint** in `snet-private-endpoints`
3. Create

📸 **Screenshot to capture:** Key Vault overview showing public access disabled and private endpoint connected. Save as `screenshots/phase-03/01-keyvault-deployed.png`.

### Section 2: Deploy the Azure AI Services Resource

1. Azure Portal → search **Azure AI services** → **+ Create** → select **Azure OpenAI**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `ai-contoso-openai`
   - Region: an Azure OpenAI-supported region matching your prior phase region where possible
   - Pricing tier: Standard S0
3. **Network tab:** Select **Disabled** for public network access; configure private endpoint in `snet-private-endpoints`, using the `privatelink.openai.azure.com` zone from Phase 2 for the DNS zone group
4. **Identity tab:** Enable **System-assigned managed identity**
5. **Encryption tab:** Select **Customer-managed keys**, point to `kv-contoso-ai`, and grant the resource's managed identity **Key Vault Crypto Service Encryption User** on the vault
6. Review + create

📸 **Screenshot to capture:** Azure AI services resource overview showing public network access disabled and managed identity enabled. Save as `screenshots/phase-03/02-ai-service-deployed.png`.

### Section 3: Deploy a Model and Configure Content Filters

1. Open **Azure AI Foundry** (or Azure OpenAI Studio) for the resource
2. **Deployments** → **+ Create new deployment** → choose a model (e.g., `gpt-4o-mini` for cost control)
3. Under **Content filter**, apply the default or a custom filter — do not deploy with filtering disabled
4. Deploy

📸 **Screenshot to capture:** Model deployment page showing the content filter applied. Save as `screenshots/phase-03/03-model-deployment-content-filter.png`.

### Section 4: Assign Azure RBAC Roles

1. `rg-secure-ai-prod` → **Access control (IAM)** → **+ Add role assignment**
   - Role: **Contributor** → Assign to: `AI-Admins` group
2. `ai-contoso-openai` resource → **Access control (IAM)** → **+ Add role assignment**
   - Role: **Cognitive Services OpenAI Contributor** → Assign to: `AI-Developers` group
   - Role: **Cognitive Services OpenAI User** → Assign to: `AI-Users` group

📸 **Screenshot to capture:** Access control (IAM) → Role assignments tab showing all three group-to-role mappings. Save as `screenshots/phase-03/04-rbac-assignments.png`.

### Section 5: Verify Private Connectivity and DNS Resolution

1. From a VM in `snet-ai-workload` (or via Bastion), attempt to resolve the AI resource's endpoint hostname — it should resolve to a private `10.1.1.x` address, not a public IP
2. Confirm the resource is unreachable from outside the VNet (attempt from a non-peered network, expect failure/timeout)

📸 **Screenshot to capture:** DNS resolution result (nslookup/Resolve-DnsName output) showing the private IP. Save as `screenshots/phase-03/05-private-dns-resolution.png`.

### Completion Checklist

- [ ] Key Vault deployed with public access disabled, connected via private endpoint
- [ ] Azure AI services resource deployed with public network access disabled
- [ ] System-assigned managed identity enabled on the AI resource
- [ ] Customer-managed key encryption configured, using Key Vault
- [ ] Model deployed with content filtering applied
- [ ] RBAC roles assigned: AI-Admins → Contributor (resource group), AI-Developers → Cognitive Services OpenAI Contributor, AI-Users → Cognitive Services OpenAI User
- [ ] Private DNS resolution verified from inside the VNet
- [ ] Public network access confirmed blocked from outside the VNet
- [ ] All screenshots captured and saved to `screenshots/phase-03/`

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
