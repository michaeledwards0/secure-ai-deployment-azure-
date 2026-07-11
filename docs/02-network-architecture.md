<div align="center">

# Phase 2: Network Architecture & Isolation
### Hub-Spoke Network Foundation for the AI Workload — Contoso AI Labs

</div>

---

## Overview

With identity established in Phase 1, Phase 2 builds the network layer that will house the AI workload. This phase implements a **hub-spoke topology** — the standard Microsoft Cloud Adoption Framework pattern for isolating workloads while centralizing shared security services (bastion access now, additional shared services later) in one place.

By the end of this phase, the AI workload will have no public network exposure: administrative access happens exclusively through Azure Bastion (no public IPs on any VM), traffic between tiers is restricted by NSGs at every subnet boundary, and every subnet's traffic is logged for observability. The network is peered but segmented — the spoke can be isolated from the hub without rebuilding anything, and the hub can support additional spokes later without touching this one.

**Environment:** Personal Azure tenant (`contosoailabs.onmicrosoft.com`) | **Duration:** ~2-3 hours | **Standard:** SC-500 blueprint, CIS Azure Foundations Benchmark v2.0 — Network Security

### Design Rationale

**Why hub-spoke instead of a single flat VNet:** A flat network means any compromised resource can potentially reach any other resource. Hub-spoke enforces segmentation by design — the hub holds shared services (Bastion now, potentially a firewall or additional shared tooling later), while the spoke holds the actual AI workload resources. Peering is explicit and one-to-one, not transitive, so a future second spoke can't reach this one without a deliberate peering decision.

**Why one spoke, not several:** This environment has a single workload — the AI service being built in Phase 3 — so a single spoke is the correct scope, not a simplification of it. The value of hub-spoke isn't the number of spokes; it's that the hub/spoke boundary exists at all. A second spoke (for example, a future data platform workload) could peer into the same hub without any change to this one, which is the actual point of the pattern. Building a second spoke with nothing to put in it would add complexity without a second workload to justify it.

**Why Azure Bastion instead of public IPs + RDP/SSH:** Public IPs on VMs are a top scanning target — internet-wide port scanners find exposed RDP/SSH within minutes. Bastion provides browser-based access to VMs over TLS through the Azure portal, with no public IP ever assigned to the VM itself and no inbound ports opened on the VM's NSG.

**Why NSGs at every subnet, not just the perimeter:** This is defense in depth applied to networking — sometimes called micro-segmentation. If one subnet is compromised, NSGs at each subnet boundary limit lateral movement to adjacent subnets rather than the entire VNet being flat and trusted.

**Why NSG flow logs, deployed now:** A network with no traffic visibility is only as trustworthy as its rules are correct — if an NSG rule is misconfigured, flow logs are what surfaces that instead of the mistake going unnoticed. Enabling flow logs in this phase, rather than waiting for Phase 5's Sentinel workspace, means network-layer observability exists from the moment the network itself exists, rather than leaving a gap between when the network is built and when it's actually being watched.

**Why Azure Firewall was evaluated and deliberately not deployed:** A centralized network firewall (for egress filtering, threat intelligence feeds, and forced tunneling) is the natural next step in a hub-spoke design and was seriously considered for this phase. It was scoped out specifically because of cost structure, not lack of awareness: Azure Firewall bills a fixed hourly fee for every hour it's deployed — starting around $0.395/hr on the Basic tier (roughly $288/month if left running continuously) — regardless of whether it processes any traffic, which doesn't fit a $50/month lab budget as an always-on resource. In a production environment with a real budget, this would be the next control added to this hub.

> **Cost note — Azure Bastion has no pause state.** Unlike a VM, Bastion has no reliable "stopped/deallocated" state — it bills continuously from creation until deletion, regardless of tier. The practical approach for a lab budget: leave Bastion running during an active work session (a few hours costs well under $1 at Basic tier's ~$0.19/hr), then delete the `bas-hub` resource when done for the session, and recreate it next time. Deleting Bastion doesn't affect the VNet, subnets, or NSGs it depends on — only the Bastion host and its public IP need to be recreated. If a Deny-Public-IP policy exclusion is scoped by resource path (subscription/resource group/resource name) rather than an internal object ID, recreating the public IP under the identical name (`pip-bastion`) in the same resource group will continue matching the existing exclusion automatically — no policy changes needed between sessions.

---

## Case Study

### Objective
Build a segmented, least-exposure, and observable network foundation for the Contoso AI Labs environment prior to deploying any AI workload resources, ensuring no compute or service in this environment is reachable from the public internet, that administrative access requires no open inbound ports, and that traffic across every subnet boundary is logged.

### Approach
*[Fill in after execution — describe your reasoning for subnet sizing, why Bastion was chosen over alternatives, how the NSG rules map to least privilege, and any tradeoffs you made between the CIDR plan and future phase needs.]*

### Controls Implemented
- Hub-spoke VNet topology with explicit, non-transitive peering
- Dedicated subnets for Bastion, shared services, and workload resources — no shared/general-purpose subnet
- Network Security Groups applied at every subnet boundary, default-deny with explicit allow rules
- NSG flow logs enabled across all three NSGs for network traffic observability, staged for Traffic Analytics once the Phase 5 Log Analytics workspace exists
- Azure Bastion for zero-public-IP administrative access, operated on a deploy/delete cost discipline given its continuous billing model
- Azure Policy assignment denying public IP creation across the subscription, with a name-based exclusion for the Bastion public IP
- DNS zone groundwork for private endpoints (used starting Phase 3, when Azure OpenAI and Key Vault are deployed)
- Single-spoke design deliberately scoped to the current workload, with the hub structured to support additional spokes without modification

### Frameworks Applied
- Microsoft Cloud Adoption Framework — Hub-Spoke Network Topology
- CIS Microsoft Azure Foundations Benchmark v2.0 — Section 6 (Networking)
- NIST 800-207 (Zero Trust Architecture) — network micro-segmentation principle

### Evidence

| Control | What it proves | Screenshot |
|---|---|---|
| Hub VNet created | `vnet-hub` provisioned with AzureBastionSubnet and snet-shared-services | ![Hub VNet](https://github.com/user-attachments/assets/c4d7378e-d0be-47b9-bd6d-d8a4cb9aad40) |
| Spoke VNet created | `vnet-spoke-ai` provisioned with snet-ai-workload and snet-private-endpoints, correctly sized at 10.1.0.0/16 | ![Spoke VNet](https://github.com/user-attachments/assets/925ad0c6-80c9-4bc5-bab5-f23d33b6c1c5) |
| VNet peering connected | Bidirectional peering between hub and spoke confirmed Connected on both sides | ![Peering](https://github.com/user-attachments/assets/c7f3f9f1-9475-48d4-a634-6d3beb125aaf) |
| NSGs configured | All three NSGs created and associated to their respective subnets | ![NSGs](https://github.com/user-attachments/assets/9fe7da23-68c0-43e9-890d-f265c498b38d) |
| NSG flow logs enabled | Flow logging active across all three NSGs, writing to dedicated storage | ![Flow logs](https://github.com/user-attachments/assets/219c4443-2b7b-4fa3-b08f-23b98260cd99) |
| Azure Bastion deployed | Bastion host running in the hub, providing zero-public-IP administrative access | ![Bastion](https://github.com/user-attachments/assets/ebb5c4ef-ac1e-4035-b1f4-c2669e73ffcc) |
| Private DNS zone established | `privatelink.openai.azure.com` created and linked to the spoke VNet, staged for Phase 3 | ![Private DNS zone](https://github.com/user-attachments/assets/86f0deee-79f1-41e4-963a-239ada16441b) |
| Deny public IP policy | Policy assignment denying public IP creation at the resource group, with a durable name-based exclusion for Bastion | ![Policy](https://github.com/user-attachments/assets/a43d70eb-141d-40c0-8f10-eb663211bccc) |

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
     - `AzureBastionSubnet` → `10.0.0.0/26` (this exact name is required by Azure Bastion; select **Azure Bastion** as the subnet purpose/template if offered — this may also auto-generate its NSG rules)
     - `snet-shared-services` → `10.0.1.0/24` (subnet purpose: **Default**)
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Hub VNet overview showing both subnets.

### Section 2: Create the Spoke VNet

1. **Virtual networks** → **+ Create**
2. **Basics:**
   - Resource group: `rg-secure-ai-prod`
   - Name: `vnet-spoke-ai`
   - Region: same as hub
3. **IP Addresses:**
   - Address space: `10.1.0.0/16` — double-check this isn't left on Azure's default suggestion, which can auto-fill as `10.0.0.0/16` and collide with the hub
   - Subnets:
     - `snet-ai-workload` → `10.1.0.0/24`
     - `snet-private-endpoints` → `10.1.1.0/24`
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Spoke VNet overview showing both subnets.

### Section 3: Peer the Hub and Spoke

Peering must be configured from both sides — each VNet needs its own peering resource pointing at the other.

1. `vnet-hub` → **Peerings** → **+ Add**
   - Name of peering from hub to spoke: `hub-to-spoke`
   - Remote virtual network: `vnet-spoke-ai`
   - Name of peering from spoke to hub: `spoke-to-hub`
   - Leave default traffic settings (allow forwarded traffic: off, allow gateway transit: off — no gateway exists)
2. **Add** — this creates both peering resources in one step
3. Confirm both show **Connected** status (may take a minute)

📸 **Screenshot to capture:** Peerings page showing "Connected" status on both sides.

### Section 4: Configure Network Security Groups

Build one NSG per subnet, each default-deny with only explicit, necessary allow rules.

> **Note on Service Tags:** when a rule below specifies a source or destination like `VirtualNetwork` or `AzureLoadBalancer`, these aren't top-level dropdown options in the Azure portal. Select **Service Tag** as the Source/Destination *type* first — a secondary field then appears where you choose the specific tag (`VirtualNetwork`, `AzureLoadBalancer`, `Internet`, etc.).

**4.1 — NSG for snet-ai-workload**

1. **Network security groups** → **+ Create**
   - Name: `nsg-ai-workload`
   - Resource group: `rg-secure-ai-prod`
2. After creation, go to **Settings → Inbound security rules** → **+ Add**:
   - **Allow-Bastion-Inbound**: Source = `10.0.0.0/26` (AzureBastionSubnet), Destination port = `3389, 22`, Priority `100`, Action = Allow
   - Leave the default **DenyAllInbound** rule (priority 65500) in place — this is what makes the subnet default-deny
3. Associate to subnet: NSG's own page → **Settings → Subnets** → **+ Associate** → Virtual network: `vnet-spoke-ai` → Subnet: `snet-ai-workload` → **OK**

**4.2 — NSG for snet-private-endpoints**

1. Create `nsg-private-endpoints`
2. Inbound rule: **Allow-VNet-Inbound** — Source type: **Service Tag** → Source service tag: **VirtualNetwork**; Destination type: **Service Tag** → Destination service tag: **VirtualNetwork**; Port = 443, Priority `100`, Action = Allow (private endpoints only need to accept traffic from within the trusted network)
3. Associate to `vnet-spoke-ai` / `snet-private-endpoints`, same Associate flow as above

**4.3 — NSG for snet-shared-services**

1. Create `nsg-shared-services`
2. Inbound rule: **Allow-Bastion-Inbound** — Source = `10.0.0.0/26`, Port = `3389, 22`, Priority `100`, Action = Allow
3. Associate to `vnet-hub` / `snet-shared-services`

> **Note:** `AzureBastionSubnet` has its own Microsoft-managed requirements for NSG rules (specific inbound rules for the Bastion control plane on ports 443, 4443, and outbound rules for session traffic). If you attach an NSG to this subnet, follow Microsoft's current published Bastion NSG requirements exactly — misconfiguring this one can break Bastion entirely.

📸 **Screenshot to capture:** Network Security Groups list showing all three NSGs with their associated subnets.

### Section 5: Enable NSG Flow Logs

Flow logs record what traffic each NSG actually allowed or denied — the difference between assuming your rules are correct and being able to prove it.

1. Deploy a storage account to hold the logs: **Storage accounts** → **+ Create**
   - Name: `stcontosoaiflowlogs` (must be globally unique — adjust if taken)
   - Resource group: `rg-secure-ai-prod`
   - Redundancy: LRS (sufficient for a lab)
   - If prompted for a **Primary service** / storage account type selector during creation, choose **Azure Blob Storage or Azure Data Lake Storage Gen 2** — flow logs write JSON output to blob containers, and the other options (Azure Files, Queue, Table) won't work for this purpose
   - **Networking tab:** disable public access if practical, or restrict to your own IP for now
2. Azure Portal → **Network Watcher** → **NSG flow logs** → **+ Create**

   The creation flow uses a unified resource picker rather than selecting one NSG directly:
   - Under **Select virtual networks to enable flow logs**, click **+ Add subnets**
   - In the **Select subnet** picker, check all three: `snet-ai-workload`, `snet-private-endpoints`, and `snet-shared-services` (these span both VNets but appear together in one list)
   - **Skip `AzureBastionSubnet`** — its control-plane traffic is already governed by Microsoft's required Bastion NSG rules, and there's little value logging it for this project
   - Confirm the selection — all three should now appear as rows in the flow log table
   - **Storage account:** confirm `stcontosoaiflowlogs` is selected (should auto-populate since it's in the same region)
   - **Retention days:** `30`
   - **Analytics tab:** leave **Traffic Analytics** disabled — it requires a Log Analytics workspace, which doesn't exist until Phase 5
   - **Review + create** → **Create**
3. Once created, spot-check by opening one of the three NSGs directly → **Settings** → look for a **Flow logs** entry confirming it's active, rather than just trusting the subnet appeared in the picker

📸 **Screenshot to capture:** NSG flow logs configuration page showing all three subnets enabled.

> **Callback to Phase 5:** once the Log Analytics workspace exists, return here and enable **Traffic Analytics** on this same flow log configuration, pointing at that workspace — this upgrades raw flow log storage into queryable, visualized traffic data alongside the Sentinel detections built in that phase.

### Section 6: Deploy Azure Bastion

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

> See the cost note in Design Rationale above — Bastion bills continuously with no pause state. Plan to delete and recreate it between work sessions rather than leaving it running idle.

📸 **Screenshot to capture:** Bastion resource overview showing Running status.

### Section 7: Establish Private DNS Zone Groundwork

Private endpoints (used starting Phase 3) require a Private DNS Zone to resolve correctly. Creating it now means Phase 3 can move straight to deployment.

1. Azure Portal → search **Private DNS zones** → **+ Create**
2. Name: `privatelink.openai.azure.com` (the standard zone name for Azure OpenAI/Cognitive Services private endpoints)
3. Resource group: `rg-secure-ai-prod`
4. **Review + create** → **Create**
5. Link the zone to the spoke VNet: zone → **Virtual network links** → **+ Add** → link to `vnet-spoke-ai`, disable auto-registration

> Note: only VNets explicitly linked to this zone resolve the private endpoint's hostname privately — peering alone does not extend this resolution. `vnet-hub` is not linked here, since nothing in the hub currently needs private access to the AI service.

📸 **Screenshot to capture:** Private DNS zone showing the virtual network link to `vnet-spoke-ai`.

### Section 8: Assign Azure Policy — Deny Public IP Creation

Extends Phase 1's governance baseline to the network layer.

1. Azure Portal → **Policy** → **Definitions** → search: `Not allowed resource types`

   *(Alternative if unavailable in your policy catalog: search `Deny public IP` or use the custom `policies/deny-public-ip.json` definition included in this repo's `policies/` folder.)*
2. **Assign** → Scope: `rg-secure-ai-prod` → configure to deny `Microsoft.Network/publicIPAddresses` creation
3. Under **Exclusions**, add the Bastion public IP's full resource path (`<subscription>/rg-secure-ai-prod/pip-bastion`) — this is a name-based path, not an internal object ID, so recreating the IP under the identical name will continue matching this exclusion in future sessions without needing to update the policy again
4. **Review + create** → **Create**

📸 **Screenshot to capture:** Policy assignment confirming the deny-public-IP rule scoped to the resource group, including the exclusion.

### Section 9: Testing & Validation

**9.1 — Verify Peering Connectivity**
`vnet-hub` → **Peerings** → confirm both peerings show **Connected**. Repeat check from `vnet-spoke-ai` → **Peerings**.

**9.2 — Verify Flow Logs Are Capturing Data**
> Azure only creates the `insights-logs-networksecuritygroupflowevent` container the first time actual flow log data is written — this requires both the flow log configuration to be fully provisioned (5-10 minutes) and real traffic passing through one of the three NSGs. With no VMs or resources deployed into these subnets yet, there may be little to nothing to capture at this point in the build. Treat full verification of this step as deferred until Phase 3 deploys a resource into `snet-ai-workload`, or until Bastion connectivity to a real VM is tested.

Storage account `stcontosoaiflowlogs` → **Containers** → check for the `insights-logs-networksecuritygroupflowevent` container once traffic exists.

**9.3 — Verify Effective NSG Rules**
Once a VM exists in `snet-ai-workload` (Phase 3+), use **Network Watcher** → **Effective security rules** to confirm only the Bastion-sourced rule and the default deny are in effect — no unexpected allow rules from platform defaults.

**9.4 — Test Bastion Connectivity**
Once a test VM exists in the spoke (can be deferred to Phase 3 if no VM exists yet): VM → **Connect** → **Bastion** → confirm browser-based session opens with no public IP required on the VM itself.

📸 **Screenshot to capture:** Flow log container contents (once available), plus effective security rules view or a successful Bastion session (whichever is available at this stage).

### Completion Checklist

- [ ] Hub VNet (`vnet-hub`) created with AzureBastionSubnet and snet-shared-services
- [ ] Spoke VNet (`vnet-spoke-ai`) created with snet-ai-workload and snet-private-endpoints, correctly sized at 10.1.0.0/16
- [ ] Bidirectional VNet peering established and confirmed Connected
- [ ] Three NSGs created and associated to their respective subnets
- [ ] AzureBastionSubnet NSG (if used) follows Microsoft's current Bastion requirements
- [ ] NSG flow logs enabled on all three NSGs, storing to the flow logs storage account
- [ ] Azure Bastion deployed and running in the hub
- [ ] Private DNS zone for `privatelink.openai.azure.com` created and linked to the spoke VNet
- [ ] Azure Policy denying public IP creation assigned to the resource group, with a durable name-based exclusion for Bastion
- [ ] Peering connectivity verified from both sides
- [ ] Flow log storage confirmed capturing data (once traffic exists)
- [ ] All screenshots captured and saved to `screenshots/phase-02/`

</details>

---

<div align="center">

[← Back to Project Overview](../README.md)

</div>
