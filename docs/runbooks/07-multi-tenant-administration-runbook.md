<div align="center">

# Phase 7 Runbook: Multi-Tenant Administration
### Step-by-Step Azure Lighthouse Delegation and Validation Guide

</div>

---

## Purpose

This runbook documents how to design, deploy, validate, and remove an Azure Lighthouse delegation between a managing tenant and a managed customer scope.

Use the accompanying [Phase 7 Portfolio Case Study](../07-multi-tenant-administration.md) for the architecture, decisions, results, and lessons learned.

---

## Scope

- Managing and managed tenant preparation
- Security-group authorization
- Role and scope design
- Registration definition
- Registration assignment
- Cross-tenant validation
- Least-privilege testing
- Activity-log review
- Offboarding

---

## Prerequisites

- Access to two Azure tenants for full validation, or a clearly documented lab limitation
- Customer-side Owner or equivalent onboarding permissions at the target scope
- Managing-tenant security group
- Tenant IDs and principal object IDs
- Azure CLI, PowerShell, portal, ARM, or Bicep deployment capability

---

## Resource Naming

| Item | Name |
|---|---|
| Managing group | `Lighthouse-Contoso-AI-Operators` |
| Offer name | `Contoso AI Managed Operations` |
| Definition name | `lighthouse-contoso-ai-operations` |
| Delegated scope | Customer resource group or subscription |
| Template folder | `lighthouse/` |

---

# 1. Define the Operating Model

Record:

- Managing tenant ID
- Managed tenant ID
- Customer subscription ID
- Delegated scope
- Authorized group object ID
- Required roles
- Customer onboarding owner
- Offboarding owner

Do not place secrets in the onboarding template.

---

# 2. Create the Managing-Tenant Security Group

1. Open **Microsoft Entra ID → Groups → New group**.
2. Create:
   - `Lighthouse-Contoso-AI-Operators`
3. Add only approved operators.
4. Record the group object ID.

## Validation

Confirm the object ID belongs to the managing tenant and not the customer tenant.

---

# 3. Select Roles and Scope

Recommended starting roles:

- Reader
- Monitoring Reader
- Security Reader

Add Contributor only for a specific controlled requirement.

Avoid delegating:

- Owner
- User Access Administrator
- Broad role-assignment control unless explicitly justified

Prefer resource-group scope for the initial demonstration.

---

# 4. Create the Registration Definition Template

Create an ARM or Bicep template containing:

- `managedByTenantId`
- Registration definition name
- Description
- Authorization entries
- Role definition IDs
- Managing-tenant principal IDs
- Principal display names

Store the files under:

```text
lighthouse/
├── main.bicep
├── main.parameters.json
└── README.md
```

Validate the template before customer deployment.

---

# 5. Deploy the Registration Definition and Assignment

From the managed customer tenant:

1. Sign in with the customer owner identity.
2. Select the target subscription or resource group.
3. Deploy the onboarding template.
4. Confirm deployment succeeds.
5. Review the created:
   - Registration definition
   - Registration assignment

## Validation

- [ ] Correct managing tenant
- [ ] Correct principal IDs
- [ ] Correct role definitions
- [ ] Correct target scope
- [ ] No unexpected authorizations

---

# 6. Validate from the Managing Tenant

1. Sign in as a member of `Lighthouse-Contoso-AI-Operators`.
2. Open **Azure Lighthouse → My customers**.
3. Confirm the customer and delegated scope are visible.
4. Open the managed resources.
5. Test permitted actions:
   - View resources
   - View monitoring data
   - View security recommendations, if delegated

---

# 7. Test Least-Privilege Boundaries

Attempt an action outside the delegated role, such as:

- Modify a resource while assigned only Reader
- Create a role assignment
- Change subscription ownership
- Access tenant-level Entra administration

Expected result: Denied or unavailable.

Capture the error as evidence of boundary enforcement.

---

# 8. Review Customer Activity Logs

From the managed tenant:

1. Open **Activity Log**.
2. Filter for actions by delegated operators.
3. Confirm operations are attributed and auditable.
4. Export relevant evidence.

---

# 9. Validate Group Membership Changes

1. Remove a test operator from the managing group.
2. Allow propagation.
3. Confirm access is removed without changing the customer registration definition.
4. Re-add only if required.

This demonstrates the operational benefit of group-based delegation.

---

# 10. Offboard the Delegation

From the managed customer tenant:

1. Locate the registration assignment.
2. Delete the assignment.
3. Confirm the managing tenant no longer sees the scope.
4. Preserve the template and evidence in source control.

> Offboarding should be controlled by the customer, not dependent on the service provider.

---

# 11. Completion Checklist

## Design

- [ ] Two-tenant scenario identified or limitation documented
- [ ] Managing group created
- [ ] Scope minimized
- [ ] Roles minimized
- [ ] Owner/User Access Administrator avoided

## Deployment

- [ ] Template created
- [ ] Parameters populated
- [ ] Registration definition deployed
- [ ] Registration assignment deployed
- [ ] Deployment evidence captured

## Validation

- [ ] Customer appears under My Customers
- [ ] Resources visible
- [ ] Permitted actions succeed
- [ ] Unauthorized actions denied
- [ ] Activity logs reviewed
- [ ] Group membership change tested

## Offboarding

- [ ] Removal procedure documented
- [ ] Assignment removal tested where practical
- [ ] Access loss verified

---

# 12. Troubleshooting

## Customer does not appear in My Customers

Check managing tenant ID, principal object ID, deployment scope, registration assignment, operator group membership, and propagation.

## Operator receives access denied for everything

Confirm the operator is in the authorized group, the group object ID is correct, and the role supports the requested read action.

## Template deployment fails

Validate tenant IDs, role definition IDs, API versions, scope, deployment permissions, and parameter formatting.

## Tenant-level Entra task is unavailable

This is expected. Azure Lighthouse delegates Azure Resource Manager scopes, not general tenant administration.

## Delegation remains visible after removal

Allow propagation, refresh tenant filters, and confirm the registration assignment—not only the definition—was removed.

---

## Related Documentation

- [Phase 7 Portfolio Case Study](../07-multi-tenant-administration.md)
- [Phase 6 — Business Continuity & Recovery](../06-business-continuity-recovery.md)
- [Phase 8 — Red Team Validation](../08-red-team-validation.md)
- [Project Overview](../../README.md)

---

<div align="center">

**Runbook complete — verify both successful delegated access and explicit denial outside the authorized scope.**

</div>
