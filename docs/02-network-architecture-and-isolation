<div align="center">

# Phase 2: Network Architecture & Isolation
### Hub-Spoke Network Foundation for the AI Workload — Contoso AI Labs

</div>

---

## Overview

With identity established in Phase 1, Phase 2 builds the network layer that will house the AI workload. This phase implements a **hub-spoke topology** — the standard Microsoft Cloud Adoption Framework pattern for isolating workloads while centralizing shared security services (bastion access, future firewall/DNS) in one place.

By the end of this phase, the AI workload will have no public network exposure: administrative access happens exclusively through Azure Bastion (no public IPs on any VM), traffic between tiers is restricted by NSGs at every subnet boundary, and the network is peered but segmented — the spoke can be isolated from the hub without rebuilding anything, and the hub can support additional spokes later without touching this one.

**Environment:** Personal Azure tenant (`contosoailabs.onmicrosoft.com`) | **Duration:** ~2-3 hours | **Standard:** SC-500 blueprint, CIS Azure Foundations Benchmark v2.0 — Network Security

### Design Rationale

**Why hub-spoke instead of a single flat VNet:** A flat network means any compromised resource can potentially reach any other resource. Hub-spoke enforces segmentation by design — the hub holds shared services (Bastion now, Azure Firewall/VPN Gateway later if needed), while the spoke holds the actual AI workload resources. Peering is explicit and one-to-one, not transitive, so a future second spoke can't reach this one without a deliberate peering decision.

**Why Azure Bastion instead of public IPs + RDP/SSH:** Public IPs on VMs are a top scanning target — internet-wide port scanners find exposed RDP/SSH within minutes. Bastion provides browser-based access to VMs over TLS through the Azure portal, with no public IP ever assigned to the VM itself and no inbound ports opened on the VM's NSG.

**Why NSGs at every subnet, not just the perimeter:** This is defense in depth applied to networking — sometimes called micro-segmentation. If one subnet is compromised, NSGs at each subnet boundary limit lateral movement to adjacent subnets rather than the entire VNet being flat and trusted.

---

## Case Study

### Objective
Build a segmented, least-exposure network foundation for the Contoso AI Labs environment prior to deploying any AI workload resources, ensuring no compute or service in this environment is reachable from the public internet and that administrative access requires no open inbound ports.

### Approach
*[Fill in after execution — describe your reasoning for subnet sizing, why Bastion was chosen over alternatives, how the NSG rules map to least privilege, and any tradeoffs you made between the CIDR plan and future phase needs.]*

### Controls Implemented
- Hub-spoke VNet topology with explicit, non-transitive peering
- Dedicated subnets for Bastion, shared services, and workload resources — no shared/general-purpose subnet
- Network Security Groups applied at every subnet boundary, default-deny with explicit allow rules
- Azure Bastion for zero-public-IP administrative access
- Azure Policy assignment denying public IP creation across the subscription
- DNS zone groundwork for private endpoints (used starting Phase 3, when Azure OpenAI and Key Vault are deployed)

### Frameworks Applied
- Microsoft Cloud Adoption Framework — Hub-Spoke Network Topology
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 6 (Networking)
- NIST 800-207 (Zero Trust Architecture) — network micro-segmentation principle

### Evidence

*[Screenshots added after execution — see capture list in the Execution Guide below.]*

### Lessons Learned
*[Fill in after execution.]*

---

## Next Phase

➡️ **[Phase 3: Azure OpenAI Deployment](./03-openai-deployment.md)**

With the network foundation in place, Phase 3 deploys Azure OpenAI into the spoke's private subnet, connected via private endpoint — no resource in this environment will have a public network path.

---

<details>
<summary><strong>📋 Full Execution Guide (click to expand)</strong> — step-by-step build instructions and completion checklist</summary>

<br>

**Duration:** ~2-3 hours
**Prerequisites:** Phase 1 complete, `rg-secure-ai-prod` resource group created, region chosen and consistent with Phase 0's Service Health alert scope

### IP Addressing Plan

Decide this before creating anything — resizing VNets after resources exist is disruptive.

| VNet | CIDR | Purpose |
|---|---|---|
| Hub VNet (`vnet-hub`) | `10.0.0.0/16` | Shared services |
| — AzureBastionSubnet | `10.0.0.0/26` | Azure Bastion (name is fixed, cannot be renamed) |
| — snet-shared-services | `10.0.1.0/24` | Future shared tooling (jump resources, monitoring agents) |
| Spoke VNet (`vnet-spoke-ai`) | `10.1.0.0/16` | AI workload |
| — snet-ai-workload | `10.1.0.0/24` | Compute resources for the AI workload |
| — snet-private-endpoints | `10.1.1.0/24` | Private endpoints for OpenAI, Key Vault, Storage (Phase 3+) |

### Section 1: Create the Hub VNet

1. Azure Portal → **Virtual networks** → **+ Create**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `vnet-hub`
   - Region: (your consistent project region)
3. **IP Addresses:**
   - Address space: `10.0.0.0/16`
   - Delete the default subnet, then add:
     - `AzureBastionSubnet` → `10.0.0.0/26` (this exact name is required by Azure Bastion)
     - `snet-shared-services` → `10.0.1.0/24`
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Hub VNet overview showing both subnets. Save as `screenshots/phase-02/01-hub-vnet-created.png`.

### Section 2: Create the Spoke VNet

1. **Virtual networks** → **+ Create**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `vnet-spoke-ai`
   - Region: same as hub
3. **IP Addresses:**
   - Address space: `10.1.0.0/16`
   - Subnets:
     - `snet-ai-workload` → `10.1.0.0/24`
     - `snet-private-endpoints` → `10.1.1.0/24`
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Spoke VNet overview showing both subnets. Save as `screenshots/phase-02/02-spoke-vnet-created.png`.

### Section 3: Peer the Hub and Spoke

Peering must be configured from both sides — each VNet needs its own peering resource pointing at the other.

1. `vnet-hub` → **Peerings** → **+ Add**
   - Name of peering from hub to spoke: `hub-to-spoke`
   - Remote virtual network: `vnet-spoke-ai`
   - Name of peering from spoke to hub: `spoke-to-hub`
   - Leave default traffic settings (allow forwarded traffic: off, allow gateway transit: off — no gateway exists yet)
2. **Add** — this creates both peering resources in one step
3. Confirm both show **Connected** status (may take a minute)

📸 **Screenshot to capture:** Peerings page showing "Connected" status on both sides. Save as `screenshots/phase-02/03-vnet-peering-connected.png`.

### Section 4: Configure Network Security Groups

Build one NSG per subnet, each default-deny with only explicit, necessary allow rules.

**4.1 — NSG for snet-ai-workload**

1. **Network security groups** → **+ Create**
   - Name: `nsg-ai-workload`
   - Resource group: `rg-secure-ai-prod`
2. After creation, add inbound rules:
   - **Allow-Bastion-Inbound**: Source = `10.0.0.0/26` (AzureBastionSubnet), Destination port = `3389, 22`, Priority `100`, Action = Allow
   - Leave the default **DenyAllInbound** rule (priority 65500) in place — this is what makes the subnet default-deny
3. Associate to subnet: **Subnets** → **Associate** → `vnet-spoke-ai` / `snet-ai-workload`

**4.2 — NSG for snet-private-endpoints**

1. Create `nsg-private-endpoints`
2. Inbound rule: **Allow-VNet-Inbound** — Source = VirtualNetwork, Destination = VirtualNetwork, Port = 443, Priority `100`, Action = Allow (private endpoints only need to accept traffic from within the trusted network)
3. Associate to `vnet-spoke-ai` / `snet-private-endpoints`

**4.3 — NSG for snet-shared-services**

1. Create `nsg-shared-services`
2. Inbound rule: **Allow-Bastion-Inbound** — Source = `10.0.0.0/26`, Port = `3389, 22`, Priority `100`, Action = Allow
3. Associate to `vnet-hub` / `snet-shared-services`

> **Note:** `AzureBastionSubnet` has its own Microsoft-managed requirements for NSG rules (specific inbound rules for the Bastion control plane on ports 443, 4443, and outbound rules for session traffic). If you attach an NSG to this subnet, follow Microsoft's current published Bastion NSG requirements exactly — misconfiguring this one can break Bastion entirely.

📸 **Screenshot to capture:** Network Security Groups list showing all three NSGs with their associated subnets. Save as `screenshots/phase-02/04-nsgs-configured.png`.

### Section 5: Deploy Azure Bastion

1. Azure Portal → search **Bastion** → **+ Create**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `bas-hub`
   - Region: same as hub
   - Tier: **Basic** (sufficient for this project; Standard adds features like native client support and shareable links not needed here)
   - Virtual network: `vnet-hub`
   - Subnet: `AzureBastionSubnet` (pre-created in Section 1)
   - Public IP: Create new → `pip-bastion`
3. **Review + create** → **Create** (takes several minutes to deploy)

📸 **Screenshot to capture:** Bastion resource overview showing Running status. Save as `screenshots/phase-02/05-bastion-deployed.png`.

### Section 6: Establish Private DNS Zone Groundwork

Private endpoints (used starting Phase 3) require a Private DNS Zone to resolve correctly. Creating it now means Phase 3 can move straight to deployment.

1. Azure Portal → search **Private DNS zones** → **+ Create**
2. Name: `privatelink.openai.azure.com` (the standard zone name for Azure OpenAI/Cognitive Services private endpoints)
3. Resource group: `rg-secure-ai-prod`
4. **Review + create** → **Create**
5. Link the zone to the spoke VNet: zone → **Virtual network links** → **+ Add** → link to `vnet-spoke-ai`, disable auto-registration

📸 **Screenshot to capture:** Private DNS zone showing the virtual network link to `vnet-spoke-ai`. Save as `screenshots/phase-02/06-private-dns-zone.png`.

### Section 7: Assign Azure Policy — Deny Public IP Creation

Extends Phase 1's governance baseline to the network layer.

1. Azure Portal → **Policy** → **Definitions** → search: `Not allowed resource types`

   *(Alternative if unavailable in your policy catalog: search `Deny public IP` or use the custom `policies/deny-public-ip.json` definition included in this repo's `policies/` folder.)*
2. **Assign** → Scope: `rg-secure-ai-prod` → configure to deny `Microsoft.Network/publicIPAddresses` creation, with an exclusion for the Bastion public IP resource already created in Section 5
3. **Review + create** → **Create**

📸 **Screenshot to capture:** Policy assignment confirming the deny-public-IP rule scoped to the resource group. Save as `screenshots/phase-02/07-deny-public-ip-policy.png`.

### Section 8: Testing & Validation

**8.1 — Verify Peering Connectivity**
`vnet-hub` → **Peerings** → confirm both peerings show **Connected**. Repeat check from `vnet-spoke-ai` → **Peerings**.

**8.2 — Verify Effective NSG Rules**
Once a VM exists in `snet-ai-workload` (Phase 3+), use **Network Watcher** → **Effective security rules** to confirm only the Bastion-sourced rule and the default deny are in effect — no unexpected allow rules from platform defaults.

**8.3 — Test Bastion Connectivity**
Once a test VM exists in the spoke (can be deferred to Phase 3 if no VM exists yet): VM → **Connect** → **Bastion** → confirm browser-based session opens with no public IP required on the VM itself.

📸 **Screenshot to capture:** Effective security rules view or a successful Bastion session (whichever is available at this stage). Save as `screenshots/phase-02/08-testing-validation.png`.

### Completion Checklist

- [ ] Hub VNet (`vnet-hub`) created with AzureBastionSubnet and snet-shared-services
- [ ] Spoke VNet (`vnet-spoke-ai`) created with snet-ai-workload and snet-private-endpoints
- [ ] Bidirectional VNet peering established and confirmed Connected
- [ ] Three NSGs created and associated to their respective subnets
- [ ] AzureBastionSubnet NSG (if used) follows Microsoft's current Bastion requirements
- [ ] Azure Bastion deployed and running in the hub
- [ ] Private DNS zone for `privatelink.openai.azure.com` created and linked to the spoke VNet
- [ ] Azure Policy denying public IP creation assigned to the resource group
- [ ] Peering connectivity verified from both sides
- [ ] All screenshots captured and saved to `screenshots/phase-02/`

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
