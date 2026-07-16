<div align="center">

# Phase 2 Runbook: Network Architecture & Isolation
### Step-by-Step Implementation Guide for the Contoso AI Labs Hub-Spoke Network

</div>

---

## Purpose

This runbook documents the implementation steps used to create the Phase 2 Azure network foundation, including hub-spoke VNets, subnets, peering, Network Security Groups, NSG flow logging, Azure Bastion, and public-IP governance.

Use the accompanying [Phase 2 Portfolio Case Study](../02-network-architecture.md) for the executive summary, architecture, engineering decisions, results, and lessons learned.

---

## Scope

This runbook covers:

- Hub and spoke VNets
- Subnet address planning
- Bidirectional VNet peering
- NSG creation and subnet association
- NSG flow logs and storage
- Azure Bastion
- Azure Policy denying public IP creation
- Validation and troubleshooting

---

## Prerequisites

- Phase 1 completed
- Azure subscription available
- Resource group `rg-secure-ai-prod`
- Consistent Azure region selected
- Permissions to create network, storage, Bastion, and policy resources
- Budget awareness for continuously billed services

**Estimated implementation time:** 2–3 hours

---

## Resource Naming

| Resource | Name |
|---|---|
| Resource group | `rg-secure-ai-prod` |
| Hub VNet | `vnet-hub` |
| AI spoke VNet | `vnet-spoke-ai` |
| Bastion | `bas-hub` |
| Bastion public IP | `pip-bastion` |
| Workload NSG | `nsg-ai-workload` |
| Private endpoint NSG | `nsg-private-endpoints` |
| Shared services NSG | `nsg-shared-services` |
| Flow log storage | `stcontosoaiflowlogs` or globally unique equivalent |

---

# 1. Plan the Address Space

Do not create the VNets until the address plan is confirmed.

| Network | CIDR | Purpose |
|---|---|---|
| `vnet-hub` | `10.0.0.0/16` | Shared services |
| `AzureBastionSubnet` | `10.0.0.0/26` | Azure Bastion |
| `snet-shared-services` | `10.0.1.0/24` | Shared tooling |
| `vnet-spoke-ai` | `10.1.0.0/16` | AI workload |
| `snet-ai-workload` | `10.1.0.0/24` | Workload compute |
| `snet-private-endpoints` | `10.1.1.0/24` | Private endpoints |

## Validation

Confirm:

- Hub and spoke address spaces do not overlap.
- Every subnet is contained within its parent VNet.
- No subnet ranges overlap.
- The Bastion subnet is named exactly `AzureBastionSubnet`.

---

# 2. Create the Hub VNet

1. Open **Azure Portal → Virtual networks**.
2. Select **Create**.
3. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `vnet-hub`
   - **Region:** Project region
4. Under **IP addresses**:
   - Set address space to `10.0.0.0/16`
   - Remove the default subnet
   - Add `AzureBastionSubnet` as `10.0.0.0/26`
   - Add `snet-shared-services` as `10.0.1.0/24`
5. Review and create.

## Validation

Open the VNet and confirm both subnets appear with the correct CIDR ranges.

**Evidence:** Capture the hub VNet subnet view.

---

# 3. Create the AI Spoke VNet

1. Open **Virtual networks → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `vnet-spoke-ai`
   - **Region:** Same as the hub
3. Under **IP addresses**:
   - Set address space to `10.1.0.0/16`
   - Add `snet-ai-workload` as `10.1.0.0/24`
   - Add `snet-private-endpoints` as `10.1.1.0/24`
4. Review and create.

> Verify that Azure did not auto-fill `10.0.0.0/16`. That would overlap with the hub and prevent peering.

## Validation

Confirm the VNet and both subnet ranges.

**Evidence:** Capture the spoke VNet subnet view.

---

# 4. Peer the Hub and Spoke

1. Open `vnet-hub`.
2. Go to **Peerings**.
3. Select **Add**.
4. Configure:
   - **Hub-side peering name:** `hub-to-spoke`
   - **Remote VNet:** `vnet-spoke-ai`
   - **Spoke-side peering name:** `spoke-to-hub`
   - Leave forwarded traffic disabled
   - Leave gateway transit disabled
5. Create the peering.

## Validation

Confirm:

- `hub-to-spoke` shows `Connected`
- `spoke-to-hub` shows `Connected`

**Evidence:** Capture the peering status.

---

# 5. Create and Associate NSGs

## 5.1 Workload NSG

1. Open **Network security groups → Create**.
2. Configure:
   - **Name:** `nsg-ai-workload`
   - **Resource group:** `rg-secure-ai-prod`
3. Create the NSG.
4. Open **Inbound security rules → Add**.
5. Configure:
   - **Name:** `Allow-Bastion-Inbound`
   - **Source:** IP Addresses
   - **Source range:** `10.0.0.0/26`
   - **Destination ports:** `22,3389`
   - **Protocol:** TCP
   - **Action:** Allow
   - **Priority:** `100`
6. Associate the NSG with:
   - **VNet:** `vnet-spoke-ai`
   - **Subnet:** `snet-ai-workload`

## 5.2 Private Endpoint NSG

1. Create `nsg-private-endpoints`.
2. Add an inbound rule:
   - **Name:** `Allow-VNet-HTTPS`
   - **Source type:** Service Tag
   - **Source:** `VirtualNetwork`
   - **Destination type:** Service Tag
   - **Destination:** `VirtualNetwork`
   - **Destination port:** `443`
   - **Protocol:** TCP
   - **Action:** Allow
   - **Priority:** `100`
3. Associate with:
   - **VNet:** `vnet-spoke-ai`
   - **Subnet:** `snet-private-endpoints`

## 5.3 Shared Services NSG

1. Create `nsg-shared-services`.
2. Add an inbound rule:
   - **Name:** `Allow-Bastion-Inbound`
   - **Source range:** `10.0.0.0/26`
   - **Destination ports:** `22,3389`
   - **Protocol:** TCP
   - **Action:** Allow
   - **Priority:** `100`
3. Associate with:
   - **VNet:** `vnet-hub`
   - **Subnet:** `snet-shared-services`

## Important Bastion Note

Do not attach a custom NSG to `AzureBastionSubnet` unless all current Microsoft-required Bastion rules are configured correctly. Incorrect rules can break the service.

## Validation

Confirm all three NSGs are associated with the intended subnets.

**Evidence:** Capture the NSG list with subnet associations.

---

# 6. Create Flow Log Storage

1. Open **Storage accounts → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `stcontosoaiflowlogs` or another globally unique name
   - **Redundancy:** LRS
   - **Primary service:** Blob Storage or Data Lake Storage Gen2, if prompted
3. Under networking:
   - Disable public access where practical, or restrict it during the lab
4. Create the storage account.

## Validation

Confirm the storage account is available in the same region as the flow-log resources where required.

---

# 7. Enable NSG Flow Logs

1. Open **Network Watcher**.
2. Go to **NSG flow logs**.
3. Select **Create**.
4. Under subnet selection, add:
   - `snet-ai-workload`
   - `snet-private-endpoints`
   - `snet-shared-services`
5. Do not include `AzureBastionSubnet`.
6. Select the flow-log storage account.
7. Set retention to `30` days.
8. Leave **Traffic Analytics** disabled until Phase 5.
9. Review and create.

## Validation

After provisioning:

1. Open each NSG.
2. Confirm the flow-log configuration is present.
3. Once traffic exists, check the storage account for the flow-event container.

> The container may not appear until real traffic passes through an NSG.

**Evidence:** Capture the flow-log configuration showing all three subnet selections.

---

# 8. Deploy Azure Bastion

1. Open **Azure Bastion → Create**.
2. Configure:
   - **Resource group:** `rg-secure-ai-prod`
   - **Name:** `bas-hub`
   - **Region:** Same as hub
   - **Tier:** Basic
   - **Virtual network:** `vnet-hub`
   - **Subnet:** `AzureBastionSubnet`
   - **Public IP:** Create `pip-bastion`
3. Review and create.

## Cost Control

Azure Bastion bills continuously and cannot be paused.

For this lab:

- Deploy it only when needed.
- Delete it between work sessions.
- Recreate it with the same resource name when required.
- Preserve the policy exclusion for `pip-bastion`.

## Validation

Confirm the Bastion resource reaches a healthy/running state.

**Evidence:** Capture the Bastion overview.

---

# 9. Assign Policy to Deny Public IP Creation

1. Open **Azure Policy → Definitions**.
2. Search for:
   - `Not allowed resource types`, or
   - a suitable built-in deny-public-IP policy, or
   - the project's custom public-IP policy
3. Assign the policy at:
   - **Scope:** `rg-secure-ai-prod`
4. Configure it to deny:
   - `Microsoft.Network/publicIPAddresses`
5. Add an exclusion for the full Bastion public-IP resource path:
   - `.../resourceGroups/rg-secure-ai-prod/providers/Microsoft.Network/publicIPAddresses/pip-bastion`
6. Review and create.

## Validation

Confirm:

- The assignment is scoped to the resource group.
- Public IP resources are denied by default.
- `pip-bastion` is excluded.
- The exclusion uses the stable resource path.

**Evidence:** Capture the policy assignment and exclusion.

---

# 10. Test and Validate the Network

## 10.1 Verify Peering

Open both VNets and confirm their peerings show `Connected`.

## 10.2 Verify NSG Associations

Confirm:

- `nsg-ai-workload` → `snet-ai-workload`
- `nsg-private-endpoints` → `snet-private-endpoints`
- `nsg-shared-services` → `snet-shared-services`

## 10.3 Verify Flow Logs

After traffic is generated:

1. Open the flow-log storage account.
2. Go to **Containers**.
3. Look for the network-security-group flow-event container.
4. Confirm log objects are being written.

## 10.4 Verify Effective Security Rules

After a VM exists in the workload subnet:

1. Open **Network Watcher**.
2. Select **Effective security rules**.
3. Choose the VM's network interface.
4. Confirm the Bastion allow rule and default deny behavior.

## 10.5 Test Bastion Connectivity

After a test VM exists:

1. Open the VM.
2. Select **Connect → Bastion**.
3. Confirm the session works.
4. Confirm the VM has no public IP.

---

# 11. Completion Checklist

## Networking

- [ ] Hub VNet created
- [ ] Hub subnets created
- [ ] Spoke VNet created
- [ ] Spoke subnets created
- [ ] Address spaces confirmed non-overlapping
- [ ] Bidirectional peering established
- [ ] Both peerings show Connected

## Security Controls

- [ ] `nsg-ai-workload` created and associated
- [ ] `nsg-private-endpoints` created and associated
- [ ] `nsg-shared-services` created and associated
- [ ] Bastion-only administrative rules configured
- [ ] Default deny behavior preserved
- [ ] No unnecessary NSG attached to AzureBastionSubnet

## Observability

- [ ] Flow-log storage account created
- [ ] Flow logs enabled for three workload-facing subnets
- [ ] Retention configured
- [ ] Traffic Analytics deferred to Phase 5
- [ ] Flow-event container verified after traffic exists

## Administration

- [ ] Azure Bastion deployed
- [ ] Bastion public IP created
- [ ] Workload VMs remain without public IPs
- [ ] Bastion cost-control process documented

## Governance

- [ ] Deny-public-IP policy assigned
- [ ] Scope set to `rg-secure-ai-prod`
- [ ] Bastion public-IP exclusion configured
- [ ] Policy assignment verified

## Evidence

- [ ] Hub VNet screenshot captured
- [ ] Spoke VNet screenshot captured
- [ ] Peering screenshot captured
- [ ] NSG screenshot captured
- [ ] Flow-log screenshot captured
- [ ] Bastion screenshot captured
- [ ] Public-IP policy screenshot captured
- [ ] Effective rules or Bastion test screenshot captured when available

---

# 12. Troubleshooting

## VNet peering fails

Check for:

- Overlapping VNet address spaces
- Incorrect subscription or tenant selection
- Missing permissions
- Existing conflicting peering configuration

## NSG rule does not work

Check:

- Rule priority
- Source CIDR
- Protocol
- Destination port
- Subnet association
- NIC-level NSGs
- Effective security rules

## Bastion deployment fails

Check:

- Subnet name is exactly `AzureBastionSubnet`
- Subnet meets the current minimum size
- Region matches the VNet
- Required public IP SKU is used
- Custom NSG rules are not blocking Bastion

## Flow-log container is missing

Check:

- Flow logs have finished provisioning
- Traffic has passed through the NSG
- Correct storage account is selected
- The selected subnet/NSG is included
- Sufficient time has passed for first log delivery

## Public IP creation is unexpectedly denied

Check:

- Policy scope
- Exclusion path
- Public IP resource name
- Whether the exclusion points to `pip-bastion`

---

## Related Documentation

- [Phase 2 Portfolio Case Study](../docs/02-network-architecture.md)
- [Phase 1 — Identity Fortress](../docs/01-identity-fortress.md)
- [Phase 3 — Azure AI Services Deployment](../docs/03-openai-deployment.md)
- [Project Overview](../README.md)

---

<div align="center">

**Runbook complete — validate every boundary before deploying the AI workload into the spoke.**

</div>
