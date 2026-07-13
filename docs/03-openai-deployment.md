<div align="center">

# Phase 3: Azure AI Services (OpenAI) Deployment
### Deploying the AI Workload — Contoso AI Labs

</div>

---

## Overview

With identity (Phase 1) and network isolation (Phase 2) in place, Phase 3 deploys the actual AI workload: an **Azure AI services** resource with the Azure OpenAI model deployment, connected exclusively through the private endpoint subnet built in Phase 2. This is also the phase where the Phase 1 identity groups stop being inert containers and become functional — Azure RBAC role assignments are applied here for the first time, connecting AI-Admins, AI-Developers, and AI-Users to real permissions on a real resource.

**Environment:** Personal Azure tenant (`contosoailabs.onmicrosoft.com`) | **Duration:** ~2-3 hours | **Standard:** SC-500 blueprint

**Business context:** the deployed model is framed as Contoso AI Labs' internal engineering assistant — intended for internal knowledge queries by employees — rather than a customer-facing product. This gives the security controls in this phase and the detections in Phase 5 a realistic purpose to defend, without requiring a custom application layer; all interaction happens through the Azure AI Foundry playground or direct API calls, consistent with the Path 1 scope decided at the start of this project.

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
*[Fill in after execution — describe your reasoning for the CMK key rotation policy, why specific guardrail categories and blocking severities were chosen, and any tradeoffs between model deployment capacity and budget constraints.]*

### Controls Implemented
- Azure AI services resource deployed with public network access disabled, reachable only via a private endpoint deployed into the Phase 2 `snet-private-endpoints` subnet
- System-assigned managed identity for Key Vault authentication
- Customer-managed keys for encryption at rest, stored in a dedicated Key Vault using the RBAC permission model, with data-plane access scoped separately to the admin account (key management) and the AI service's managed identity (wrap/unwrap operations only)
- Content filtering configured at the model deployment level
- Azure RBAC role assignments connecting Phase 1's identity groups to real permissions on the AI resource
- Private DNS zone groups created and linked to the spoke VNet, one per resource (`privatelink.vaultcore.azure.net` for Key Vault, `privatelink.cognitiveservices.azure.com` for Azure AI services)
- A disposable, out-of-architecture scaffolding resource used solely to satisfy the Foundry Project workspace layer, deliberately excluded from CMK, private endpoints, and the project's security narrative — kept isolated in its own resource group with a scoped policy exemption, distinct from the actual protected workload

### RBAC Role Assignments

| Group | Role | Scope | Assignment Type | Rationale |
|---|---|---|---|---|
| **AI-Admins** | `Contributor` | `rg-secure-ai-prod` resource group | Eligible (PIM, 3-month expiration, 4hr activation, MFA/justification/approval required) | Full administrative control to manage all resources in the environment, consistent with the group's Phase 1 "PIM required for activation" description |
| **AI-Developers** | `Cognitive Services OpenAI Contributor` | Azure AI services resource | Active (standing) | Deploy models, fine-tune, and generate content — matches the "deploy, configure, and test" scope defined in Phase 1 |
| **AI-Users** | `Cognitive Services OpenAI User` | Azure AI services resource | Active (standing) | Inference and content generation only, no deployment or configuration rights — matches "consume AI services via approved interfaces only" |

### Frameworks Applied
- Microsoft Cloud Adoption Framework — Data Encryption and Key Management
- OWASP Top 10 for LLM Applications — LLM06 (Sensitive Information Disclosure), addressed via network isolation and CMK
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 4 (Storage/Key Management analog applied to Cognitive/AI resources)
- NIST 800-207 (Zero Trust Architecture) — least-privilege authorization

### Evidence
*[Screenshots added after execution — see capture list in the Execution Guide below.]*

### Lessons Learned
*[Fill in after execution — the customer-managed key setup is a strong candidate here: configuring CMK on a fully network-locked Key Vault required working through four distinct access boundaries in sequence (network exception, personal RBAC for key management, client IP for the portal's key picker, and a separate managed-identity RBAC grant that the portal's own UI claimed would happen automatically but didn't). Worth reflecting on what this revealed about Key Vault's layered data-plane vs. management-plane access model, and why "the portal says it will handle this" shouldn't be taken as guaranteed without verification.

A second strong candidate: discovering that creating an Azure AI services resource does not automatically create a Foundry Project — the two are separate layers, and the Project-creation UI has no way to attach to an existing resource at creation time, only after. Combined with the subscription-scoped deny-public-access policy blocking the auto-provisioned scaffolding resource regardless of resource group, this required a deliberate, scoped policy exemption on a dedicated resource group — a good demonstration of understanding that Deny policies block writes into non-compliant states rather than forcing ongoing compliance, and that a narrow, reversible exception is preferable to weakening a policy's overall enforcement.

Also worth a short mention: the originally planned model, `gpt-4o-mini`, was retired by Azure mid-project (February 27, 2026), requiring a deployment target change to `gpt-5-mini`. A minor thing, but a real reminder that Azure OpenAI's model catalog turns over quickly, and a production owner of this deployment would need an operational process for tracking retirement schedules rather than treating a deployed model as a permanent fixture.

A fourth item: extending Phase 1's PIM pattern to a resource-level Azure RBAC role surfaced that PIM for Azure resources lives in a completely separate part of the portal from a resource group's own IAM blade, requiring the resource group to be explicitly discovered and onboarded first. Also worth noting the fresh subscription's zero standing compute quota, which required a quota increase request before the first VM of the project — small and quickly approved, but a good example of a platform guardrail that has nothing to do with the architecture itself and shouldn't be mistaken for a configuration mistake.]*

---

## Next Phase

➡️ **[Phase 4: Governance & Defender for Cloud](./04-governance-defender.md)**

With the workload deployed, Phase 4 layers Defender for Cloud workload protection and an expanded governance baseline over everything built so far.

---

<details>
<summary><strong>📋 Full Execution Guide (click to expand)</strong> — step-by-step build instructions and completion checklist</summary>

<br>

**Prerequisites:** Phase 1 and Phase 2 complete, `snet-private-endpoints` subnet already exists. Both DNS zones required for this phase's private endpoints — `privatelink.vaultcore.azure.net` for Key Vault and `privatelink.cognitiveservices.azure.com` for Azure AI services — are created inline during Sections 1 and 2 below, alongside the private endpoints that need them.

### Section 1: Deploy the Key Vault

1. Azure Portal → **Key Vaults** → **+ Create**
   - Resource group: `rg-secure-ai-prod`
   - Name: `kv-contoso-ai` (must be globally unique — append a suffix if needed)
   - Region: consistent with prior phases
   - Pricing tier: Standard
2. **Networking tab:** Uncheck **Enable public access**
3. Under **Private endpoint**, click **+ Create a private endpoint** and configure:
   - **Name:** `pe-kv-contoso-ai`
   - **Region:** consistent with prior phases
   - **Target sub-resource:** `vault`
   - **Virtual network:** `vnet-spoke-ai`
   - **Subnet:** `snet-private-endpoints`
   - **Private DNS integration:** enable it. The correct zone name for Key Vault is **`privatelink.vaultcore.azure.net`**, and the portal will offer to auto-create and link it to `vnet-spoke-ai` as part of this step. Let it.
   - **OK** to add the private endpoint, then continue back on the Key Vault creation wizard
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Key Vault overview showing public access disabled and private endpoint connected. Save as `screenshots/phase-03/01-keyvault-deployed.png`.

### Section 2: Deploy the Azure AI Services Resource

1. Azure Portal → search **Azure AI services** → **+ Create**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `ai-contoso-openai`
   - Region: an Azure OpenAI-supported region matching your prior phase region where possible
   - Pricing tier: Standard S0
3. **Network tab:**
   - Type: **Disabled** (no public/selected-network access)
   - Under **Private endpoint**, click **+ Add Private Endpoint** and configure:
     - **Name:** `pe-aiservices-contoso-ai`
     - **Target sub-resource:** `account` (this is the correct sub-resource for the unified Azure AI services resource type — not `openai`)
     - **Virtual network:** `vnet-spoke-ai`
     - **Subnet:** `snet-private-endpoints`
     - **Integrate with private DNS zone:** Yes
     - **Private DNS Zone:** accept the auto-suggested **`(New) cognitiveservices`** zone (creates and links `privatelink.cognitiveservices.azure.com` to `vnet-spoke-ai`)
   - **OK**, then continue back on the Azure AI services creation wizard
4. **Identity tab:** Set **System assigned managed identity → Status: On**. Leave User assigned managed identity empty — system-assigned is what the rest of this project relies on.
5. **Tags tab:** skip, or add tags consistent with your other resources
6. **Review + create** → **Create** — note there is no dedicated Encryption tab during creation for this unified resource type; customer-managed key configuration happens after deployment (Section 2b below)

📸 **Screenshot to capture:** Azure AI services resource overview showing public network access disabled and managed identity enabled. Save as `screenshots/phase-03/02-ai-service-deployed.png`.

### Section 2b: Configure Customer-Managed Keys (Post-Deployment)

Unlike the creation wizard, which has no Encryption tab for this resource type, CMK is configured directly on the deployed resource. Because `kv-contoso-ai` has public network access fully disabled (Phase 3, Section 1), this step involves working through several distinct access boundaries in sequence — network, two separate RBAC grants, and client IP — not a single Save action. Each layer below produced its own distinct portal error before being resolved; they are documented in the order actually encountered.

**1. Temporarily open the vault's network for CMK provisioning**

The vault's "Disabled" public access state blocks even Azure's own internal CMK provisioning call — the "Allow trusted Microsoft services" exception only takes effect when the vault is in "Enabled from selected networks" mode, not "Disabled."

1. `kv-contoso-ai` → **Settings → Networking**
2. Change **Public network access** from **Disabled** to **Enabled from selected virtual networks and IP addresses**
3. Leave the virtual network / IP allow list empty for now — this does not reopen the vault to arbitrary public traffic
4. Under **Exceptions**, set **Allow trusted Microsoft services to bypass this firewall** to **Yes**
5. **Save** and allow a couple of minutes to propagate

**2. Grant yourself Key Vault data-plane access**

Being Contributor/Owner on the resource group does not grant access to keys/secrets inside a Key Vault using the RBAC permission model — data-plane access is separate and must be granted explicitly.

1. `kv-contoso-ai` → **Access control (IAM)** → **+ Add role assignment**
2. **Role:** `Key Vault Crypto Officer`
3. **Assign access to:** your own admin account
4. **Review + assign**, then wait 2-5 minutes for propagation

**3. Allow your own client IP through the firewall**

The "Select a key" picker queries the vault directly from your browser, separate from the trusted-services exception above — it needs your current public IP explicitly allowed.

1. `kv-contoso-ai` → **Settings → Networking** → **Firewall**
2. Add your current client IP (the portal typically offers an auto-detect/add option)
3. **Save**, allow a minute to propagate

**4. Create the key**

No key exists in the vault yet at this point.

1. Back on `ai-contoso-openai` → **Encryption** → **Select from Key Vault** → **Select a KeyVault and Key for encryption**
2. **Key vault:** `kv-contoso-ai` → **Create new key**
3. **Options:** Generate | **Name:** `key-contoso-ai-cmk` | **Key type:** RSA | **RSA key size:** 2048
4. Leave activation/expiration dates unset for now; optionally configure a **key rotation policy** (e.g., 12 months) as a good practice detail even without executing a rotation
5. **Create**, then select the new key and its version back on the "Select a key" screen

**5. Grant the managed identity access, and save**

The Encryption blade's own UI states the resource's managed identity "will be granted access to the selected key vault" automatically — in practice this did not hold, and the Save attempt failed with an access-forbidden error referencing wrap/unwrap key operations. The grant has to be made manually:

1. `kv-contoso-ai` → **Access control (IAM)** → **+ Add role assignment**
2. **Role:** `Key Vault Crypto Service Encryption User`
3. **Assign access to:** **Managed identity** → **+ Select members** → identity type **Azure AI services** (may display as **Cognitive Services**) → select `ai-contoso-openai`
4. **Review + assign**, wait 2-5 minutes for propagation
5. Return to `ai-contoso-openai` → **Encryption** (the vault/key selection should still be populated) → **Save** — this should now succeed

**6. Decide the vault's steady-state network posture**

CMK operations (rotation, re-wrap) are infrequent once initial setup is complete. Consider reverting `kv-contoso-ai`'s Public network access back to **Disabled** now that CMK is configured, removing your own client IP from the firewall allow list, and reopening this same sequence only when a future key rotation is needed — keeping the vault's default posture as closed as possible rather than permanently leaving the "Selected networks" exception open.

📸 **Screenshot to capture:** Encryption blade showing customer-managed key configuration successfully saved, pointing to `kv-contoso-ai` / `key-contoso-ai-cmk`. Save as `screenshots/phase-03/02b-cmk-configured.png`. Consider also capturing the IAM role assignments list on `kv-contoso-ai` showing both roles granted (yourself: Key Vault Crypto Officer; managed identity: Key Vault Crypto Service Encryption User), since this is strong evidence of the layered access model in action.

### Section 3: Create a Foundry Project and Deploy a Model

**Important context before starting:** creating `ai-contoso-openai` as a standalone Azure AI services resource (Section 2) does **not** automatically create a Foundry Project. A Project is a separate workspace layer on top of the resource — it's what actually exposes the Playground, model deployment UI, agents, and evaluations. The resource holds the infrastructure (keys, networking, identity); the Project is where you interact with it. This distinction isn't obvious from the resource's own Overview page, which has no direct link into Foundry.

**3.1 — Attempt to create a Project, expect a policy conflict**

1. Go to **ai.azure.com** → **Management Center** (or the "+ New project" option if surfaced directly)
2. **Project name:** `contoso-ai-project`
3. Under **Advanced options**, set **Resource group** to `rg-secure-ai-prod` and confirm the region matches your other resources
4. In the **Microsoft Foundry resource** field, this dialog does **not** support attaching an existing resource — it always tries to provision a new one. Typing the name of your existing `ai-contoso-openai` will only return a naming-collision error, not an option to reuse it.
5. Give the auto-provisioned resource a distinct name (e.g., `foundry-contoso-ai-workspace`) and click **Create**
6. **Expect this to fail** with an error like *"Resource 'foundry-contoso-ai-workspace' was disallowed by Deny Public Access on Azure AI Services."* This is Phase 3's own governance policy correctly blocking a new, publicly-accessible AI resource — the policy is assigned at **subscription scope**, so it applies regardless of which resource group the Project creation flow targets, not just `rg-secure-ai-prod`.

**3.2 — Grant a scoped policy exemption for the scaffolding resource**

Rather than weakening the policy itself, exclude only this specific resource group from it — Deny policies block resources from being *created into* a non-compliant state; they don't force a resource to remain non-compliant once it exists, which is what makes this approach safe.

1. Azure Portal → **Policy** → **Assignments** → open **"Deny Public Access on Azure AI Services"**
2. Add a **Policy exemption** (or edit the assignment's exclusions) scoped to a dedicated resource group — e.g., `rg-foundry-scaffolding` — created specifically to hold this disposable Project-supporting resource, kept separate from `rg-secure-ai-prod`
3. Save, then retry the Project creation dialog from 3.1, targeting `rg-foundry-scaffolding` this time — it should now succeed

**3.3 — Lock down the scaffolding resource's networking (without CMK or a private endpoint)**

The auto-created resource (`foundry-contoso-ai-workspace`) is disposable plumbing that exists only so the Foundry workspace has something to anchor to — it is **not** part of the secured architecture and is never referenced as a protected asset in this project's case study. It doesn't need CMK or a private endpoint in `snet-private-endpoints`; that level of hardening is reserved for `ai-contoso-openai`, the actual workload. It does still need enough network configuration to not sit fully open and to avoid re-triggering the same access issues seen with Key Vault's CMK setup:

1. Open the new resource → **Networking**
2. Set **Public network access** to **Selected networks and private endpoints**, leaving the network rules list empty (not adding any VNets or IPs)
3. Enable **"Allow Azure services on the trusted services list to bypass this firewall"**
4. Save — avoid "Disabled," which (per the Key Vault CMK experience in Section 2b) can block Microsoft's own internal service calls even with a trusted-services exception configured, since that exception only takes effect in "Selected networks" mode

**3.4 — Connect the real resource to the Project**

1. Inside the newly created Project, go to **Management Center** → look for **Connected resources** (or **Connections**)
2. **+ New connection** → **Azure OpenAI**

   > **Known limitation:** the "Azure OpenAI" connection type's resource picker filters specifically for the older, dedicated Azure OpenAI resource kind — it may show **no resources available**, even though `ai-contoso-openai` exists and is fully deployed, because it was created as the newer unified **Azure AI services** kind. If the dropdown comes up empty, this is a resource-kind mismatch in the connection UI, not a missing or misconfigured resource.
   >
   > **Workaround:** Also confirm there's no **native "Model deployments" blade directly on `ai-contoso-openai`'s own Azure Portal page** (checked under Resource Management, alongside Keys and Endpoint / Encryption) — if present, deploying directly there sidesteps the Foundry connection issue entirely. If it's not present for this resource kind, fall back to a manual connection: choose **Custom keys** (or **API Key**) as the connection type instead of "Azure OpenAI," and manually supply the **Endpoint** and a **Key** copied from `ai-contoso-openai` → **Keys and Endpoint**. This creates a working connection regardless of the resource-kind filtering limitation.
3. Confirm the Project's model deployment interface can now target `ai-contoso-openai` specifically, rather than only the scaffolding resource

📸 **Screenshot to capture:** the policy exemption scoped to `rg-foundry-scaffolding`, and the Project's Connected resources list showing `ai-contoso-openai` attached. Save as `screenshots/phase-03/03a-foundry-project-connected.png`.

**3.5 — Deploy the model and configure guardrails**

Content filtering is now presented as **Guardrails** — a broader, named set of risk categories rather than a single filter toggle.

1. In the Project, confirm the active/target resource is `ai-contoso-openai`
2. **Deployments** → **+ Create new deployment** → choose a lightweight, cost-effective model — `gpt-4o-mini` was originally used here but was retired by Azure on February 27, 2026; **`gpt-5-mini`** is the current equivalent-tier replacement (verify it's listed in your region's model catalog before deploying, since availability shifts over time)
3. Under **Guardrails**, configure:
   - **Jailbreak:** keep enabled, action **Block** — this maps directly to Phase 6's jailbreak red team test
   - **Indirect prompt injections:** enable this (off by default) — maps directly to Phase 5's prompt injection detection rule and Phase 6's prompt injection test; leave **Spotlighting (Preview)** off, not required for this project's scope
   - **Content harms** (Hate, Sexual, Self-harm, Violence): leave at default **Medium blocking**, User input + Output, Block; leave the **Profanity** blocklist enabled
   - **Protected materials** (code and text): keep enabled — reasonable default for an internal engineering assistant
   - **Sensitive data leakage → PII (Preview):** keep enabled, and select at least one PII data type (e.g., Email Address, Phone Number) — this field requires a selection to save, and pairs well with the Purview DSPM for AI addition planned for Phase 4
   - **Task drift:** leave unchecked/off — this control applies to agentic tool-calling workflows, which are out of scope for this project's Path 1 deployment (playground/API only, no agent)
4. Save the guardrail configuration, then **Deploy**

📸 **Screenshot to capture:** the full Guardrails configuration page (both scroll positions covering all categories), and the Model deployment page confirming it's deployed against `ai-contoso-openai`. Save as `screenshots/phase-03/03-model-deployment-guardrails.png`.

### Section 4: Assign Azure RBAC Roles

Consistent with Phase 1's design — where **AI-Admins** was explicitly scoped as *"Full administrative access to AI workload — PIM required for activation"* — the Contributor assignment for AI-Admins continues that Zero Trust, just-in-time pattern at the resource group level. AI-Developers and AI-Users were never scoped as requiring elevation, so their roles are assigned as standing (Active) access instead.

**4.1 — Assign AI-Admins → Contributor (Eligible)**

1. `rg-secure-ai-prod` → **Access control (IAM)** → **+ Add role assignment**
2. Search for **Contributor**. If it doesn't appear under the default role list, check the **Privileged administrator roles** tab (as opposed to **Job function roles**) — Contributor and Owner are grouped there in the current portal UI, not in the default search results.
3. **Members:** assign to `AI-Admins` group
4. **Assignment type:** **Eligible**
5. **Assignment duration:** if given a choice beyond "Permanent eligible," set a fixed end date roughly 3 months out, matching Phase 1's Section 6.2 precedent. If the assignment defaults to **Permanent eligible**, this needs to be corrected — standing eligibility that never expires undermines least-privilege even though it isn't standing *active* access. Go back and edit the assignment afterward if this happens (see 4.2, which also fixes it going forward at the policy level).

**4.2 — Configure activation requirements via Privileged Identity Management**

The eligible assignment alone only means the role *can* be activated — MFA, justification, and approval requirements are configured separately, on the resource group's PIM policy for this role. PIM for Azure resources isn't found under the resource group's own IAM blade; it's a separate top-level service that first needs the resource group onboarded.

1. Search the top bar for **Privileged Identity Management** → open it
2. Left sidebar → **Manage** → **Azure resources**
3. Drill down the scope selectors: **Management groups** → `Contoso AI Labs` → **Subscriptions** → `Azure subscription 1` → **Resource groups** → `rg-secure-ai-prod`. Leave **Resources** empty — you're managing the resource group as a whole, not one resource inside it.
4. Confirm **Current selection** shows `rg-secure-ai-prod`, Type: Resource group
5. Click **Manage resource**
6. Left sidebar → **Roles** → find **Contributor** → **Settings** → **Edit**
7. On the **Activation** tab, mirror Phase 1's Section 6.2 settings: **Activation maximum duration: 4 hours**; require **Microsoft Entra Multifactor Authentication**; require **justification**; require **ticket information**; require **approval to activate**, approver: yourself
8. On the **Assignment** tab: set **Allow permanent eligible assignment** to **Off**, and **Expire eligible assignments after** to **3 months** — this also corrects the assignment going forward if it was initially created as Permanent in Step 4.1
9. **Update**

**4.3 — Assign AI-Developers and AI-Users (Active, standing access)**

1. `ai-contoso-openai` resource → **Access control (IAM)** → **+ Add role assignment**
2. Role: **Cognitive Services OpenAI Contributor** → Assign to: `AI-Developers` group → **Assignment type: Active** (not Eligible — this group has standing access by design, matching its Phase 1 description of deploy/configure/test access)
3. Role: **Cognitive Services OpenAI User** → Assign to: `AI-Users` group → **Assignment type: Active**

📸 **Screenshot to capture:** Access control (IAM) → Role assignments tab showing all three group-to-role mappings (noting AI-Admins as Eligible, the other two as Active), plus the PIM Contributor role settings page showing the activation requirements. Save as `screenshots/phase-03/04-rbac-assignments.png` and `screenshots/phase-03/04b-pim-contributor-settings.png`.

### Section 5: Deploy a Test VM and Verify Private Connectivity

Neither Phase 2 nor Phase 3 up to this point actually deploys anything into `snet-ai-workload` — Bastion has had nothing to connect to, and there's been no compute inside the VNet to test DNS resolution from. This section closes that gap with a small, disposable test VM.

**5.1 — Deploy the test VM**

1. Azure Portal → **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `vm-ai-workload-test`
   - Region: consistent with prior phases
   - Image: **Windows Server 2022 Datacenter** (or 2025 — either works identically for this test), specifically the **[smalldisk] ... Gen 2** variant — smaller default disk, current-generation UEFI boot
   - Size: smallest available, e.g. **Standard_B2s**. If your subscription has zero standing compute quota (common for a subscription that's never deployed a VM before), size selection will prompt a **quota increase request** — this is a platform-level anti-abuse guardrail unrelated to anything in this project's configuration. Requesting a small amount (e.g., 4 vCPUs) is typically approved within minutes.
   - Authentication type: Password, for simplicity on a short-lived test VM — set a strong username/password and record it
3. **Disks tab:** defaults are fine (Standard SSD)
4. **Networking tab:**
   - Virtual network: `vnet-spoke-ai`
   - Subnet: `snet-ai-workload`
   - **Public IP: None**
   - **Public inbound ports: None** — no RDP/SSH rule should ever be opened directly on this VM; Bastion is the only access path, via the existing `Allow-Bastion-Inbound` rule on `nsg-ai-workload` from Phase 2
   - Confirm the NIC inherits `nsg-ai-workload` from the subnet rather than creating a new NSG
5. **Review + create** → **Create**

> **Expected warning:** once deployed, the VM's Overview page may show *"VM agent status is not ready"* for a few minutes after first boot. This is normal — the guest agent takes time to initialize and does not block Bastion connectivity, which operates at the network/RDP layer independently of agent status. Refresh after a few minutes and it typically clears on its own.

📸 **Screenshot to capture:** VM Overview showing Running status, no public IP, and the correct private IP within `10.1.0.0/24`. Save as `screenshots/phase-03/05a-test-vm-deployed.png`.

**5.2 — Connect via Bastion and test DNS resolution**

1. VM → **Connect** → **Bastion** → select `bas-hub` → enter the VM's credentials → **Connect**
2. Once inside the VM's desktop session, open **PowerShell**
3. Run: `Resolve-DnsName <ai-contoso-openai endpoint hostname>` (copy the exact hostname from `ai-contoso-openai` → **Keys and Endpoint**)
4. Confirm the result resolves to a private address inside `10.1.1.0/24` (the `snet-private-endpoints` range) — not a public IP

📸 **Screenshot to capture:** PowerShell output showing the private IP resolution. Save as `screenshots/phase-03/05-private-dns-resolution.png`.

**5.3 — Confirm no public path exists**

Attempt to resolve or reach the same hostname from a network outside the VNet (e.g., your own machine, not through Bastion) — expect failure or timeout, confirming the resource has no public network path.

📸 **Screenshot to capture:** the failed/timeout resolution attempt from outside the VNet, for contrast against the successful private resolution above.

### Completion Checklist

- [ ] Key Vault deployed with public access disabled, connected via private endpoint
- [ ] Azure AI services resource deployed with public network access disabled
- [ ] System-assigned managed identity enabled on the AI resource
- [ ] Key Vault firewall temporarily opened to Selected networks with trusted-services exception enabled
- [ ] Key Vault Crypto Officer role granted to admin account for key management access
- [ ] Client IP added to Key Vault firewall for key picker access
- [ ] Encryption key created in Key Vault (RSA 2048)
- [ ] Key Vault Crypto Service Encryption User role granted to the AI service's managed identity
- [ ] Customer-managed key encryption configured and saved successfully
- [ ] Key Vault network posture reverted to a tighter steady state post-configuration
- [ ] Foundry Project created, with a scoped policy exemption for the disposable scaffolding resource group
- [ ] Scaffolding resource networking locked to Selected networks + trusted services (no CMK, no private endpoint — intentionally out of scope)
- [ ] `ai-contoso-openai` connected to the Foundry Project as a resource connection (via Azure OpenAI connection type, or Custom Keys fallback if the resource-kind mismatch blocks the picker)
- [ ] Model deployed with Guardrails configured (Jailbreak, Indirect prompt injections, Content harms, Protected materials, Sensitive data leakage with at least one PII type selected)
- [ ] AI-Admins → Contributor assigned as Eligible at `rg-secure-ai-prod`, with PIM activation requirements configured (4hr max, MFA, justification, ticket info, approval) and permanent eligible assignment disabled (3-month expiration)
- [ ] AI-Developers → Cognitive Services OpenAI Contributor and AI-Users → Cognitive Services OpenAI User assigned as Active (standing) access on `ai-contoso-openai`
- [ ] Test VM (`vm-ai-workload-test`) deployed into `snet-ai-workload` with no public IP and no public inbound ports
- [ ] Private DNS resolution verified from inside the VM, resolving to a `10.1.1.0/24` address
- [ ] Public network access confirmed blocked when tested from outside the VNet
- [ ] All screenshots captured and saved to `screenshots/phase-03/`

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
