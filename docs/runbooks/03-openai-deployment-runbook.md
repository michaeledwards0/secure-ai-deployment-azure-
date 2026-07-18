<div align="center">

# Phase 3 Runbook: Azure AI Services Deployment
### Step-by-Step Implementation Guide for the Contoso AI Labs AI Workload

</div>

---

## Purpose

This runbook documents the implementation steps used to deploy and secure the Phase 3 Azure AI workload, including Key Vault, private endpoints, private DNS, managed identity, customer-managed keys, Azure AI Foundry, guardrails, RBAC, PIM, and private-connectivity validation.

Use the accompanying [Phase 3 Portfolio Case Study](../03-openai-deployment.md) for the executive summary, architecture, engineering decisions, results, and lessons learned.

---

## Scope

This runbook covers:

- Key Vault deployment
- Azure AI Services deployment
- Private endpoints and private DNS
- System-assigned managed identity
- Customer-managed key configuration
- Azure AI Foundry project setup
- Model deployment and guardrails
- Azure RBAC and PIM
- Test VM deployment
- Private DNS and network validation

---

## Prerequisites

- Phase 1 complete
- Phase 2 complete
- `rg-secure-ai-prod` exists
- `vnet-spoke-ai` exists
- `snet-private-endpoints` exists
- `snet-ai-workload` exists
- Azure Bastion available for testing
- Permissions to create AI, Key Vault, networking, role, and policy resources

---

## Resource Naming

| Resource | Name |
|---|---|
| Key Vault | `kv-contoso-ai` |
| Key Vault private endpoint | `pe-kv-contoso-ai` |
| AI resource | `ai-contoso-openai` |
| AI private endpoint | `pe-aiservices-contoso-ai` |
| CMK | `key-contoso-ai-cmk` |
| Foundry project | `contoso-ai-project` |
| Foundry scaffolding RG | `rg-foundry-scaffolding` |
| Test VM | `vm-ai-workload-test` |

---

# 1. Deploy Key Vault

1. Open **Azure Portal → Key Vaults → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `kv-contoso-ai` or globally unique equivalent
   - **Region:** Project region
   - **Pricing tier:** Standard
   - **Permission model:** Azure RBAC
3. Under **Networking**:
   - Disable public access
   - Create private endpoint `pe-kv-contoso-ai`
   - Target sub-resource: `vault`
   - VNet: `vnet-spoke-ai`
   - Subnet: `snet-private-endpoints`
   - Enable private DNS integration
   - Use `privatelink.vaultcore.azure.net`
4. Review and create.

## Validation

Confirm:

- Key Vault deployed
- Public access disabled
- Private endpoint approved
- Private DNS zone linked to `vnet-spoke-ai`

---

# 2. Deploy Azure AI Services

1. Open **Azure AI Services → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `ai-contoso-openai`
   - **Region:** Supported project region
   - **Pricing tier:** Standard S0
3. Under **Networking**:
   - Set public network access to Disabled
   - Create private endpoint `pe-aiservices-contoso-ai`
   - Target sub-resource: `account`
   - VNet: `vnet-spoke-ai`
   - Subnet: `snet-private-endpoints`
   - Enable private DNS integration
   - Use `privatelink.cognitiveservices.azure.com`
4. Under **Identity**:
   - Enable system-assigned managed identity
5. Review and create.

## Validation

Confirm:

- Public access disabled
- Managed identity enabled
- Private endpoint approved
- Private DNS zone linked

---

# 3. Prepare Key Vault for CMK Provisioning

## 3.1 Temporarily Adjust Network Access

1. Open `kv-contoso-ai`.
2. Go to **Networking**.
3. Change public network access from Disabled to:
   - **Enabled from selected virtual networks and IP addresses**
4. Enable:
   - **Allow trusted Microsoft services to bypass this firewall**
5. Save.

## 3.2 Grant Administrator Data-Plane Access

1. Open **Access control (IAM)** on the Key Vault.
2. Add role:
   - `Key Vault Crypto Officer`
3. Assign it to the administrator account.
4. Wait for propagation.

## 3.3 Add Client IP

1. Return to **Key Vault → Networking**.
2. Add the current public client IP.
3. Save.
4. Wait for propagation.

---

# 4. Create the Customer-Managed Key

1. Open `ai-contoso-openai`.
2. Go to **Encryption**.
3. Select **Customer-managed key**.
4. Choose **Select from Key Vault**.
5. Select `kv-contoso-ai`.
6. Create a new key:
   - **Name:** `key-contoso-ai-cmk`
   - **Type:** RSA
   - **Size:** 2048
7. Configure a rotation policy if desired:
   - Example: 12 months
8. Select the key and version.

---

# 5. Grant the AI Managed Identity Key Access

1. Open `kv-contoso-ai`.
2. Go to **Access control (IAM)**.
3. Add role:
   - `Key Vault Crypto Service Encryption User`
4. Assign access to:
   - Managed identity
   - Azure AI Services / Cognitive Services
   - `ai-contoso-openai`
5. Wait for propagation.
6. Return to the AI resource's **Encryption** blade.
7. Save the CMK configuration.

## Validation

Confirm the encryption blade shows:

- Key Vault name
- Key name
- Key version
- Successful save

---

# 6. Restore Key Vault Steady-State Security

After CMK configuration:

1. Remove the administrator's temporary client IP.
2. Review whether trusted-services access remains required.
3. Return public network access to Disabled where operationally possible.
4. Save.
5. Confirm the private endpoint remains approved.

---

# 7. Create the Azure AI Foundry Project

## 7.1 Create the Project

1. Open Azure AI Foundry.
2. Create project:
   - **Name:** `contoso-ai-project`
3. Note that the project creation flow may attempt to provision a new Foundry resource.

## 7.2 Handle Policy Conflict

If the deployment is denied by the project's public-access policy:

1. Create `rg-foundry-scaffolding`.
2. Open the relevant Azure Policy assignment.
3. Create a narrowly scoped exemption or exclusion for this resource group.
4. Retry the project creation in `rg-foundry-scaffolding`.

Do not weaken the subscription-wide policy.

## 7.3 Lock Down the Scaffolding Resource

1. Open the auto-created resource.
2. Set networking to:
   - Selected networks and private endpoints
3. Leave the allow list empty.
4. Enable trusted Microsoft service bypass if required.
5. Do not treat this scaffolding resource as the protected AI workload.

---

# 8. Connect the Protected AI Resource

1. Open the Foundry project.
2. Go to **Management Center → Connected resources**.
3. Add an Azure OpenAI / Azure AI connection.
4. Select `ai-contoso-openai`.

If the picker does not show the resource:

- Check for direct model deployment on the Azure resource
- Use a custom endpoint/key connection only as a portal compatibility fallback
- Keep `ai-contoso-openai` as the actual protected workload

## Validation

Confirm the project can target the intended AI resource.

---

# 9. Deploy the Model and Configure Guardrails

1. Open **Deployments → Create deployment**.
2. Choose an available cost-effective model:
   - Example: `gpt-5-mini`
3. Configure Guardrails:

## Jailbreak

- Enabled
- Action: Block

## Indirect Prompt Injection

- Enabled
- Action: Block
- Spotlighting: Off unless explicitly testing it

## Content Harms

For hate, sexual, self-harm, and violence:

- Threshold: Medium
- User input: Block
- Output: Block

## Protected Materials

- Code: Enabled
- Text: Enabled

## Sensitive Data Leakage

- PII: Enabled
- Select at least one data type such as:
  - Email address
  - Phone number

## Task Drift

- Leave off because agentic tool-calling is outside scope

4. Save guardrails.
5. Deploy the model.

## Validation

Confirm the deployment is associated with `ai-contoso-openai` and the guardrails are active.

---

# 10. Assign Azure RBAC Roles

## 10.1 AI-Admins

1. Open `rg-secure-ai-prod → Access control (IAM)`.
2. Add role:
   - Contributor
3. Assign to:
   - `AI-Admins`
4. Assignment type:
   - Eligible

## 10.2 Configure PIM

1. Open **Privileged Identity Management**.
2. Go to **Azure resources**.
3. Select:
   - Management Group
   - Subscription
   - Resource group `rg-secure-ai-prod`
4. Manage the resource.
5. Open **Roles → Contributor → Settings**.
6. Configure:
   - Maximum activation: 4 hours
   - Require MFA
   - Require justification
   - Require ticket information
   - Require approval
   - Approver: administrator
7. Disable permanent eligible assignments.
8. Set eligibility expiration to 3 months.
9. Save.

## 10.3 AI-Developers

Assign:

- Role: `Cognitive Services OpenAI Contributor`
- Scope: `ai-contoso-openai`
- Group: `AI-Developers`
- Assignment: Active

## 10.4 AI-Users

Assign:

- Role: `Cognitive Services OpenAI User`
- Scope: `ai-contoso-openai`
- Group: `AI-Users`
- Assignment: Active

## Validation

Confirm all three mappings and assignment types.

---

# 11. Deploy the Test VM

1. Open **Virtual machines → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `vm-ai-workload-test`
   - **Image:** Windows Server 2022 or later
   - **Size:** Smallest practical size
   - **Authentication:** Strong local password
3. Under **Networking**:
   - VNet: `vnet-spoke-ai`
   - Subnet: `snet-ai-workload`
   - Public IP: None
   - Public inbound ports: None
   - Use subnet-level `nsg-ai-workload`
4. Review and create.

## Validation

Confirm:

- VM running
- Private IP in `10.1.0.0/24`
- No public IP
- No public inbound rules

---

# 12. Validate Private DNS and Connectivity

## 12.1 Connect Through Bastion

1. Open the VM.
2. Select **Connect → Bastion**.
3. Use the local VM credentials.
4. Open PowerShell inside the session.

## 12.2 Resolve the AI Endpoint

Run:

```powershell
Resolve-DnsName <ai-service-endpoint-hostname>
```

Expected result:

- Private address in `10.1.1.0/24`
- No public endpoint returned for the protected path

## 12.3 Test Outside the VNet

From a device outside the VNet:

- Attempt to reach the AI endpoint
- Expect failure, denial, or timeout based on service behavior

---

# 13. Completion Checklist

## Key Vault

- [ ] Key Vault deployed
- [ ] RBAC permission model enabled
- [ ] Public access disabled in steady state
- [ ] Private endpoint connected
- [ ] Private DNS configured
- [ ] CMK created
- [ ] Rotation policy considered or configured
- [ ] Admin assigned Crypto Officer
- [ ] AI managed identity assigned Crypto Service Encryption User

## Azure AI Services

- [ ] AI resource deployed
- [ ] Public access disabled
- [ ] System-assigned identity enabled
- [ ] Private endpoint connected
- [ ] Private DNS configured
- [ ] CMK saved successfully

## Foundry and Model

- [ ] Foundry project created
- [ ] Policy exemption narrowly scoped
- [ ] Protected AI resource connected
- [ ] Model deployed
- [ ] Jailbreak protection enabled
- [ ] Indirect prompt-injection protection enabled
- [ ] Content-harm filters configured
- [ ] Protected materials enabled
- [ ] PII guardrail configured
- [ ] Task drift left off by design

## RBAC and PIM

- [ ] AI-Admins assigned Contributor as Eligible
- [ ] PIM maximum activation set to 4 hours
- [ ] MFA required
- [ ] Justification required
- [ ] Ticket information required
- [ ] Approval required
- [ ] Eligibility expires after 3 months
- [ ] AI-Developers assigned OpenAI Contributor
- [ ] AI-Users assigned OpenAI User

## Connectivity

- [ ] Test VM deployed
- [ ] VM has no public IP
- [ ] VM has no public inbound ports
- [ ] Bastion connectivity tested
- [ ] AI endpoint resolves to `10.1.1.x`
- [ ] Public path tested and blocked

## Evidence

- [ ] Key Vault screenshot captured
- [ ] AI Services screenshot captured
- [ ] CMK screenshot captured
- [ ] Foundry connection screenshot captured
- [ ] Guardrails screenshot captured
- [ ] RBAC screenshot captured
- [ ] PIM screenshot captured
- [ ] Test VM screenshot captured
- [ ] Private DNS screenshot captured

---

# 14. Troubleshooting

## Key picker cannot access the vault

Check:

- Current client IP is allowed
- Administrator has `Key Vault Crypto Officer`
- Public network access is set to selected networks during provisioning
- Trusted Microsoft services bypass is enabled if needed
- RBAC changes have propagated

## CMK save fails

Check:

- AI resource has system-assigned identity
- Managed identity has `Key Vault Crypto Service Encryption User`
- Key is enabled
- Vault and AI resource meet regional/service requirements
- Key Vault networking allows the required provisioning path

## Foundry project creation fails

Check:

- Azure Policy denial details
- Resource-group scope
- Whether the auto-created resource violates public-access requirements
- Whether a narrow exemption is appropriate

## AI resource does not appear in Foundry

Check:

- Resource type compatibility
- Available connection types
- Direct deployment options
- Endpoint/key fallback only as a portal compatibility workaround

## Private DNS resolves publicly

Check:

- Correct private DNS zone
- VNet link exists
- Private endpoint DNS zone group exists
- VM uses Azure-provided DNS or the correct custom resolver
- Endpoint hostname is correct

## Bastion cannot connect to the VM

Check:

- VM is running
- NSG allows Bastion subnet on the required port
- No public IP is required
- Correct local credentials are used
- VNet peering is Connected

---

## Related Documentation

- [Phase 3 Portfolio Case Study](../03-openai-deployment.md)
- [Phase 2 — Network Architecture & Isolation](../02-network-architecture.md)
- [Phase 4 — Governance & Defender for Cloud](../04-governance-defender.md)
- [Project Overview](../../README.md)

---

<div align="center">

**Runbook complete — verify the effective encryption, identity, guardrail, and network state before moving into continuous monitoring.**

</div>
