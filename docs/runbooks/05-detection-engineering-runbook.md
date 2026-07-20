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
- Four telemetry-supported analytics rules
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
- `AzureDiagnostics` for Azure AI administrative telemetry
- `AzureMetrics` where supported
- `SecurityAlert` or Defender-related tables where available

Generate benign test activity if required.

## Validation

- [ ] Identity logs present
- [ ] Azure Activity present, if subscription Activity Log export is configured
- [ ] Azure AI administrative telemetry present in `AzureDiagnostics`
- [ ] Timestamps are current
- [ ] Workspace time range includes test events

> `search *` only displays tables that contain records in the selected time range. A configured connector or diagnostic setting does not guarantee that a table will appear until a new matching event is generated.

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

# 5. Build the Off-Hours Azure AI Administrative Change Rule

The available Azure AI telemetry in this lab contained administrative `Audit` events rather than model-inference request content. Therefore, the original **Off-Hours AI Access** idea was revised to detect administrative changes made to the Azure AI resource outside approved hours.

1. Open **Sentinel → Analytics → Create → Scheduled query rule**.
2. Name: `Off-Hours Azure AI Administrative Change`.
3. Severity: Low or Medium.
4. Use the Azure AI administrative records available in `AzureDiagnostics`.
5. Convert UTC timestamps to the intended local time zone.
6. Detect events outside approved business hours and on weekends.
7. Map the Azure resource and initiating account only when those fields are present.
8. Enable incident creation and test with a benign configuration change.

Example pattern:

```kusto
AzureDiagnostics
| where Category == "Audit"
| where OperationName has_any ("UpdateResource", "Vnet")
| extend LocalTime = datetime_utc_to_local(TimeGenerated, "US/Central")
| extend LocalHour = hourofday(LocalTime), DayOfWeek = dayofweek(LocalTime)
| where LocalHour < 7 or LocalHour >= 19 or DayOfWeek in (time(0.00:00:00), time(6.00:00:00))
| project TimeGenerated, LocalTime, OperationName, ResourceId, ResultType, CallerIpAddress
```

> Adjust the operation names, hours, projected fields, and entity mappings to match the columns present in the workspace.

---

# 6. Build the Privileged Group Membership Change Rule

This rule replaced unsupported prompt-content detections. It uses Microsoft Entra `AuditLogs`, which were available after diagnostic settings were configured and fresh audit activity was generated.

1. Generate a controlled event by adding or removing a test user from a privileged test group.
2. Confirm the event appears in `AuditLogs`.
3. Open **Sentinel → Analytics → Create → Scheduled query rule**.
4. Name: `Privileged Group Membership Change`.
5. Severity: Medium or High, depending on group scope.
6. Filter for group membership add/remove operations.
7. Scope the rule to approved privileged groups where possible.
8. Map the initiating account, target account, and IP address only when the query exposes them.
9. Enable incident creation and configure grouping by group or target account.

Example discovery pattern:

```kusto
AuditLogs
| where OperationName has_any ("Add member to group", "Remove member from group")
| project TimeGenerated, OperationName, InitiatedBy, TargetResources, Result, CorrelationId
| order by TimeGenerated desc
```

> Parse `InitiatedBy` and `TargetResources` only after confirming their structure in the tenant. Do not map fields that are not present in the final query output.

---

# 7. Deferred Detections Due to Missing Telemetry

The following planned detections were removed from Phase 5 because the required telemetry was not available in the workspace:

- **Anomalous Token Consumption** — no Azure OpenAI token-consumption metrics were present in `AzureMetrics`.
- **Prompt Injection and Jailbreak Detection** — prompt text, request bodies, and content-filter classifications were not being ingested.

These detections must not be claimed as implemented. Revisit them only after enabling a telemetry source that exposes the required fields. Prompt-injection and jailbreak validation may be addressed during Phase 8 red-team testing.

---

# 8. Build Impossible-Travel Correlation

1. Use successful `SigninLogs` records with user, time, and location data.
2. Scope the rule to identities that can reach or administer the AI workload.
3. Compare each sign-in with the previous successful sign-in for the same user.
4. Calculate elapsed time, geographic distance, and estimated travel speed.
5. Consider VPN, mobile carrier, and incomplete geolocation data as expected false-positive sources.
6. Run the complete query as one selection in the Logs editor. Running only part of a multi-statement query can cause errors such as unresolved variables or missing tabular expressions.
7. Start in low-volume testing mode and enable only after the query runs successfully against actual tenant fields.

> An empty result is valid and means no impossible-travel pattern was found during the selected time range.

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
3. Use the **Microsoft Sentinel incident** trigger.
4. Publish or save the Logic App before attempting to attach it to an automation rule.
5. Use a managed identity where supported.
6. Add conditions:
   - Incident created by the intended rule
   - Account equals the dedicated test identity
   - Account is not an emergency, administrator, or service account
7. Actions:
   - Add an incident comment
   - Notify the administrator only after configuring the required Outlook or Teams API connection
   - Optionally disable the dedicated test identity
8. For **Add comment to incident**, manually enter or bind the incident ARM ID as required by the current portal experience.
9. Save and publish the workflow.
10. Grant the playbook identity only the permissions required.

> Outlook and Teams actions require separate API connections. They may be omitted during the first validation so the Sentinel trigger and incident-comment action can be tested independently.

> Keep account disablement limited to a dedicated test identity until the logic is proven.

---

# 11. Create the Automation Rule

1. Open **Sentinel → Automation**.
2. Create a **Standard** automation rule. Do not select an Enhanced Rule for this workflow.
3. Trigger: Incident created.
4. Condition: Analytics rule name matches the approved high-confidence rule.
5. Action: **Run playbook**. Do not use **Run generated playbook**, which only lists AI-generated playbooks.
6. Select `auto-contain-ai-test-user`.
7. If the playbook does not appear, use **Manage playbook permissions** and grant Microsoft Sentinel the required Automation Contributor access to the resource group containing the Logic App.
8. Enable the automation rule.

---

# 12. Validate Conditional Access Insights

1. Open **Microsoft Entra ID → Conditional Access → Insights and reporting**.
2. Select `law-contoso-ai` if prompted.
3. Confirm the embedded workbook loads. It opens directly inside the Entra Conditional Access blade rather than as a separate workbook page.
4. Select **User sign-ins**.
5. Change the workspace filter from **All** to `law-contoso-ai` when available.
6. Expand the time range, such as **Last 7 days**, if the default 24-hour window has little data.
7. Review the Impact Summary and policy-result details.

> A result of **Not applied** still confirms that the workbook is functioning; it means no enabled Conditional Access policy applied to that sign-in. The warning about `ServicePrincipalSignInLogs` affects only the service-principal tab.

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
- [ ] Off-hours Azure AI administrative-change rule created
- [ ] Privileged group membership-change rule created
- [ ] Impossible-travel rule validated or documented as producing no results
- [ ] Token-consumption detection documented as deferred due to missing telemetry
- [ ] Prompt-injection and jailbreak detections documented as deferred due to missing telemetry
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

## Playbook does not appear in the automation rule

Confirm the Logic App is saved or published, create a **Standard** automation rule, choose **Run playbook** rather than **Run generated playbook**, and configure **Manage playbook permissions** for the resource group.

## Add Comment to Incident cannot resolve the incident ID

Use the incident ARM ID expected by the Sentinel connector. In some portal experiences this value must be entered or bound manually rather than selected from Dynamic Content.

## Outlook or Teams action requests a connection

These actions require an API connection. Configure the connection explicitly or omit the notification action during the initial playbook test.

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

**Runbook complete — only enable detections supported by telemetry actually present in the workspace.**

</div>
