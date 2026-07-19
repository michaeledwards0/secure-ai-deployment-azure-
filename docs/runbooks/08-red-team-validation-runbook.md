<div align="center">

# Phase 8 Runbook: Red Team Validation
### Authorized Control Testing, Findings, and Retesting Guide

</div>

---

## Purpose

This runbook defines a controlled, authorized process for testing AI guardrails, Azure RBAC, PIM, Sentinel detections, and response automation within the Contoso AI Labs environment.

Use the accompanying [Phase 8 Portfolio Case Study](../08-red-team-validation.md) to record final findings and lessons.

---

## Scope and Authorization

Testing is limited to:

- Azure tenants and subscriptions owned by the user
- Dedicated test identities
- The Contoso AI Labs resources
- Benign prompts and synthetic data
- Approved Sentinel and automation tests

Do not test third-party systems or introduce real confidential data.

---

## Prerequisites

- Phases 1–7 complete
- Phase 5 rules enabled and schema-validated
- Dedicated test identity
- Test account excluded from emergency and production administration
- Automation limited to the test identity
- Screenshots and incident exports ready for evidence collection

---

## Test Record Template

For each test record:

| Field | Value |
|---|---|
| Test ID | |
| Date/time | |
| Operator | |
| Identity used | |
| Objective | |
| Preconditions | |
| Exact action | |
| Expected preventive result | |
| Expected detection result | |
| Actual result | |
| Incident ID/reference | |
| Finding | |
| Remediation | |
| Retest result | |

---

# 1. Pre-Test Safety Review

- [ ] Confirm resource ownership
- [ ] Confirm test identity
- [ ] Confirm no real secrets are present
- [ ] Confirm automation exclusions
- [ ] Confirm rollback steps
- [ ] Confirm Sentinel ingestion
- [ ] Record baseline resource state

---

# 2. Prompt-Injection Indicator Test

1. Use the approved Azure AI playground or test client.
2. Submit a benign instruction-override string.
3. Record:
   - Model response
   - Guardrail response
   - Request timestamp
4. Search relevant logs.
5. Check Sentinel for the intended rule.
6. Document whether:
   - Prevented
   - Logged
   - Detected
   - Incident created

Do not assume prompt text is available in logs unless verified.

---

# 3. Jailbreak-Indicator Test

1. Use a harmless synthetic jailbreak-style string.
2. Confirm content-safety behavior.
3. Record timestamp and test identity.
4. Review Sentinel.
5. Confirm automation does not affect any account other than the test identity.
6. Record the result.

---

# 4. Sensitive-Information Disclosure Test

1. Use a synthetic system prompt or dummy secret.
2. Request protected instructions or data.
3. Confirm the model does not expose real confidential information.
4. Record refusal, guardrail, logging, and detection results.
5. Create a finding if the model discloses the synthetic protected value unexpectedly.

---

# 5. Unauthorized Azure Access Test

1. Sign in as a user without the required AI Contributor role.
2. Attempt a read or modification action that exceeds the assigned role.
3. Confirm a 403 or equivalent denial.
4. Review Azure Activity and Sentinel.
5. Record whether the denied action produced usable telemetry.

---

# 6. Privilege-Escalation Boundary Test

1. Use a developer-level identity.
2. Attempt an administrator-only resource action.
3. Confirm Azure RBAC denial.
4. Attempt the legitimate PIM activation workflow.
5. Confirm MFA, justification, approval, and duration requirements.
6. Do not bypass or weaken PIM controls for testing.

---

# 7. Detection and Response Test

1. Choose a previously unit-tested rule.
2. Generate its approved benign trigger.
3. Wait for the schedule.
4. Confirm:
   - Alert
   - Incident
   - Entity mappings
   - Automation-rule evaluation
   - Playbook action
5. Verify only the test identity is affected.
6. Restore the test identity if disabled.

---

# 8. Analyze Findings

Classify each outcome:

- Pass
- Partial pass
- Fail
- Not testable with current telemetry
- False positive
- False negative

Assign severity based on impact and exploitability, not cosmetic preference.

---

# 9. Remediate and Retest

Potential remediation areas:

- Diagnostic settings
- KQL table or field selection
- Thresholds
- Entity mappings
- Incident grouping
- Guardrail settings
- RBAC scope
- PIM configuration
- Automation exclusions

Repeat the original test exactly and record the new result.

---

# 10. Completion Checklist

- [ ] Authorization documented
- [ ] Safety review complete
- [ ] Prompt-injection test executed
- [ ] Jailbreak-indicator test executed
- [ ] Disclosure test executed
- [ ] Unauthorized-access test executed
- [ ] PIM/RBAC boundary test executed
- [ ] Detection-response test executed
- [ ] Findings classified
- [ ] Remediations assigned
- [ ] Retests completed where applicable
- [ ] Evidence sanitized
- [ ] Test identities restored

---

# 11. Troubleshooting

## No detection fires

Confirm the query returns the event, the rule is enabled, schedule and lookback overlap, required fields exist, and ingestion completed.

## Guardrail blocks but no Sentinel event exists

Document prevention as successful and detection as unsupported or failed. Review whether the service emits the needed telemetry.

## Playbook affects the wrong identity

Stop the automation, restore the identity, review entity mappings and conditions, and restrict the playbook to the dedicated test user.

## Access test unexpectedly succeeds

Immediately stop, capture evidence, review effective RBAC and PIM assignments, remediate excessive access, and retest.

---

## Related Documentation

- [Phase 8 Portfolio Case Study](../08-red-team-validation.md)
- [Phase 7 — Multi-Tenant Administration](../07-multi-tenant-administration.md)
- [Phase 9 — Infrastructure as Code](../09-infrastructure-as-code.md)
- [Project Overview](../../README.md)

---

<div align="center">

**Runbook complete — record observed results honestly and retest every material remediation.**

</div>
