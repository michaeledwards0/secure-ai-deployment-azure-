<div align="center">

# Phase 5 Runbook: Detection Engineering
### Step-by-Step Microsoft Sentinel, KQL Analytics, and Automation Guide

</div>

---

## Purpose

This runbook documents how to enable Microsoft Sentinel on the Phase 4 Log Analytics Workspace, validate data ingestion, create and tune analytics rules, configure entity mappings, build controlled automation, and perform an end-to-end incident test.

Use the accompanying [Phase 5 Portfolio Case Study](../05-detection-engineering.md) for the architecture, decisions, results, and lessons learned.

---

## Scope

This runbook covers:

- Microsoft Sentinel onboarding
- Data-source validation
- KQL schema discovery
- Six analytics rules
- Entity mapping
- Incident grouping
- Logic App automation
- Conditional Access reporting
- End-to-end testing
- Evidence collection and troubleshooting

---

## Prerequisites

- Phases 1–4 complete
- `law-contoso-ai` receives identity and Azure resource telemetry
- Microsoft Entra diagnostic settings configured
- Azure AI Services diagnostics configured
- Permissions to manage Sentinel, analytics rules, Logic Apps, and automation rules
- Dedicated test identity available for safe validation

---

## Resource Naming

| Resource | Name |
|---|---|
| Log Analytics Workspace | `law-contoso-ai` |
| Sentinel solution | Microsoft Sentinel on `law-contoso-ai` |
| Playbook | `auto-contain-ai-test-user` |
| Automation rule | `run-ai-containment-playbook` |
| KQL folder | `kql/` |

---

# 1. Enable Microsoft Sentinel

1. Open **Azure Portal → Microsoft Sentinel**.
2. Select **Create**.
3. Choose `law-contoso-ai`.
4. Select **Add**.
5. Wait for onboarding to complete.

## Validation

Confirm:

- The workspace appears under Microsoft Sentinel
- Overview and Analytics pages load
- No second workspace was created unnecessarily

---

# 2. Validate Data Ingestion

Open **Sentinel → Logs** and run:

```kusto
search *
| summarize Events=count() by $table
| order by Events desc
```

Check for expected tables such as:

- `SigninLogs`
- `AuditLogs`
- `AzureActivity`
- `AzureDiagnostics`
- Resource-specific Azure AI tables
- `SecurityAlert` or Defender-related tables where available

Generate benign test activity if required.

## Validation

- [ ] Identity logs present
- [ ] Azure Activity present
- [ ] AI/resource telemetry present
- [ ] Timestamps are current
- [ ] Workspace time range includes test events

---

# 3. Create a Query-Validation Worksheet

For every planned rule, record:

| Field | Value |
|---|---|
| Data table | |
| Required columns | |
| Lookback period | |
| Query frequency | |
| Threshold | |
| Entity mappings | |
| Expected false positives | |
| Test method | |

Do not activate a rule until its source table and fields are confirmed.

---

# 4. Build the Break-Glass Sign-In Rule

1. Open **Sentinel → Analytics → Create → Scheduled query rule**.
2. Name: `Break-Glass Account Sign-In`.
3. Severity: High.
4. Query example:

```kusto
SigninLogs
| where UserPrincipalName =~ "breakglass@contosoailabs.onmicrosoft.com"
| project TimeGenerated, UserPrincipalName, IPAddress, Location, ResultType, ResultDescription
```

5. Configure:
   - Query frequency: 5–15 minutes
   - Lookup period: slightly longer than frequency
   - Alert threshold: Greater than 0
6. Map:
   - Account → `UserPrincipalName`
   - IP → `IPAddress`
7. Create incidents from alerts.
8. Enable the rule.

> Do not perform an unnecessary real sign-in to the emergency account solely for testing. Use a controlled equivalent test identity or temporarily clone the query logic.

---

# 5. Build the Off-Hours AI Access Rule

1. Confirm the AI access table and timestamp field.
2. Define expected business hours.
3. Create `Off-Hours AI Access`.
4. Filter events outside the approved time window.
5. Map account, IP, and resource where available.
6. Start at Low or Medium severity.
7. Test with controlled activity.

---

# 6. Build the Anomalous Token-Consumption Rule

1. Identify the usage or token-count field.
2. Establish a baseline from available data.
3. Create a query comparing recent consumption to historical behavior.
4. Use thresholds suitable for a small lab.
5. Avoid hard-coding a threshold that cannot be justified.
6. Enable after validation.

Example pattern:

```kusto
// Replace table and fields with those present in the workspace.
AzureDiagnostics
| where ResourceProvider has "COGNITIVE"
| summarize Requests=count() by bin(TimeGenerated, 15m), Resource
| order by TimeGenerated desc
```

---

# 7. Build Prompt-Injection and Jailbreak Indicator Rules

1. Confirm that request text or relevant classifications are actually logged.
2. If prompt text is unavailable, detect supported proxy indicators rather than claiming content inspection.
3. Store queries in:
   - `kql/prompt-injection-detection.kql`
   - `kql/jailbreak-attempts.kql`
4. Configure severity based on confidence.
5. Add entity mappings.
6. Enable incident creation.
7. Test using benign, authorized strings in the lab.

---

# 8. Build Impossible-Travel Correlation

1. Use `SigninLogs` location and timestamp data.
2. Scope the rule to identities that can reach or administer the AI workload.
3. Correlate geographically distant sign-ins within an unrealistic time window.
4. Consider VPN and mobile-network false positives.
5. Start in low-volume testing mode.
6. Enable after tuning.

---

# 9. Configure Entity Mappings and Incident Grouping

For each rule, map only fields that actually exist.

Recommended entities:

- Account
- IP
- Host
- Azure resource
- Cloud application

Configure incident grouping so repeated events from the same account or resource do not create excessive duplicate incidents.

## Validation

Open a test incident and confirm the Investigation graph contains useful entities.

---

# 10. Build the Logic App Playbook

1. Open **Sentinel → Automation → Create → Playbook with incident trigger**.
2. Name: `auto-contain-ai-test-user`.
3. Use a managed identity where supported.
4. Add conditions:
   - Incident created by the intended rule
   - Account equals the dedicated test identity
   - Account is not an emergency, administrator, or service account
5. Actions:
   - Notify the administrator
   - Add an incident comment
   - Optionally disable the dedicated test identity
6. Save.
7. Grant the playbook identity only the permissions required.

> Keep account disablement limited to a dedicated test identity until the logic is proven.

---

# 11. Create the Automation Rule

1. Open **Sentinel → Automation**.
2. Create an automation rule.
3. Trigger: Incident created.
4. Condition: Analytics rule name matches the approved high-confidence rule.
5. Action: Run `auto-contain-ai-test-user`.
6. Enable the automation rule.

---

# 12. Validate Conditional Access Insights

1. Open **Microsoft Entra ID → Conditional Access → Insights and reporting**.
2. Select `law-contoso-ai` if prompted.
3. Confirm the workbook loads.
4. Review policy results over the available time range.

---

# 13. Perform an End-to-End Test

1. Select a low-risk detection.
2. Generate controlled activity with the test identity.
3. Wait for the analytics-rule schedule.
4. Confirm:
   - Alert created
   - Incident created
   - Entities mapped
   - Automation rule evaluated
   - Playbook ran only when conditions matched
5. Close the incident as benign positive or test activity.
6. Record any tuning changes.

---

# 14. Completion Checklist

## Sentinel

- [ ] Sentinel enabled on `law-contoso-ai`
- [ ] Required tables verified
- [ ] Data timestamps current

## Analytics

- [ ] Break-glass rule created
- [ ] Off-hours rule created
- [ ] Token-consumption rule created
- [ ] Prompt-injection indicator rule created
- [ ] Jailbreak indicator rule created
- [ ] Impossible-travel rule created
- [ ] Entity mappings configured
- [ ] Incident grouping configured

## Automation

- [ ] Logic App created
- [ ] Managed identity permissions minimized
- [ ] Emergency/service-account exclusions configured
- [ ] Automation rule attached
- [ ] Test identity only used for disable action

## Validation

- [ ] Test alert generated
- [ ] Incident generated
- [ ] Investigation entities visible
- [ ] Playbook behavior validated
- [ ] False-positive observations documented

## Evidence

- [ ] Sentinel overview captured
- [ ] Data-ingestion evidence captured
- [ ] Workbook captured
- [ ] Analytics-rule list captured
- [ ] Entity mappings captured
- [ ] Playbook captured
- [ ] Test incident captured

---

# 15. Troubleshooting

## Query returns no data

Check the table name, time range, diagnostic settings, category selection, and whether test events exist.

## Query field does not exist

Use:

```kusto
<TableName>
| getschema
```

Then revise the rule to use fields actually present.

## Analytics rule does not fire

Check scheduling, lookback, threshold, query results, rule enabled state, and ingestion delay.

## Incident has no entities

Review entity mappings and ensure the selected columns are included in the final query projection.

## Playbook cannot disable the test user

Check managed-identity permissions, connector authentication, automation-rule conditions, and whether the account is protected by exclusions.

## Too many incidents

Increase thresholds, narrow scope, add allowlists, and configure incident grouping.

---

## Related Documentation

- [Phase 5 Portfolio Case Study](../05-detection-engineering.md)
- [Phase 4 — Governance & Defender for Cloud](../04-governance-defender.md)
- [Phase 6 — Business Continuity & Recovery](../06-business-continuity-recovery.md)
- [Project Overview](../../README.md)

---

<div align="center">

**Runbook complete — verify each rule against real workspace fields before relying on it operationally.**

</div>
