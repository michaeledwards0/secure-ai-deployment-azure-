<div align="center">

# Secure AI Deployment on Azure

### End-to-End Cloud Security Architecture for Azure OpenAI Workloads

*A Zero Trust reference implementation demonstrating how to design, deploy, defend, govern, and codify a production-grade AI workload on Microsoft Azure.*

**Environment:** Personal Azure Tenant
**Duration:** 6 Weeks  
**Author:** Michael Edwards — Cloud Security Engineer

</div>

---

## Executive Summary

**Contoso AI Labs** (fictional startup) is deploying its first Azure OpenAI service to production. As the Cloud Security Engineer, my mandate is to:

1. **Design** a Zero Trust security architecture for AI workloads
2. **Deploy** the environment with security controls baked in from day one
3. **Govern** the environment through policy as code
4. **Defend** the workload with layered detection and response
5. **Validate** the defenses through red team simulation
6. **Codify** the entire deployment as Infrastructure as Code for reproducibility

This repository documents the full engagement — from policy authoring through post-deployment attack validation and IaC handoff — in the same format a real Cloud Security Engineer would use to hand off work to leadership and future engineers.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENTRA ID IDENTITY LAYER                      │
│  Conditional Access · MFA · PIM · Named Locations · Break-Glass │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                   NETWORK ISOLATION LAYER                       │
│    Hub-Spoke VNet · Private Endpoints · NSGs · Azure Bastion    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     AI WORKLOAD LAYER                           │
│   Azure OpenAI · Managed Identity · Content Filters · Key Vault │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     GOVERNANCE LAYER                            │
│    Azure Policy · Compliance Baseline · Guardrails · Audit      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                  WORKLOAD PROTECTION LAYER                      │
│    Defender for Cloud · Defender for AI Services · Secure Score │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                DETECTION & RESPONSE LAYER                       │
│    Microsoft Sentinel · Custom Analytics Rules · Playbooks      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                RED TEAM VALIDATION LAYER                        │
│  Prompt Injection · Jailbreak · Exfiltration · Access Attempts  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              INFRASTRUCTURE AS CODE (BICEP)                     │
│    Modular Templates · Parameters · Deployment Automation       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Project Phases

| Phase | Focus Area | Status |
|:---:|---|:---:|
| **[Phase 1](./docs/01-identity-fortress.md)** | Identity Fortress — Entra ID, Conditional Access, PIM, Azure Policy (baseline) | 🟡 In Progress |
| **[Phase 2](./docs/02-network-architecture.md)** | Network Architecture — VNets, Private Endpoints, NSGs, Bastion | ⚪ Pending |
| **[Phase 3](./docs/03-openai-deployment.md)** | Azure OpenAI Deployment — Managed Identity, CMK, Content Filters | ⚪ Pending |
| **[Phase 4](./docs/04-governance-defender.md)** | Governance & Defender for Cloud — Azure Policy, Workload Protection | ⚪ Pending |
| **[Phase 5](./docs/05-detection-engineering.md)** | Detection Engineering — Sentinel Rules & Playbooks | ⚪ Pending |
| **[Phase 6](./docs/06-red-team-findings.md)** | Red Team Validation — Attack Simulation & Findings | ⚪ Pending |
| **[Phase 7](./docs/07-infrastructure-as-code.md)** | Infrastructure as Code — Bicep Templates & Deployment | ⚪ Pending |

---

## Repository Structure

```
secure-ai-deployment-azure/
├── README.md                                (this file)
├── docs/
│   ├── 01-identity-fortress.md
│   ├── 02-network-architecture.md
│   ├── 03-openai-deployment.md
│   ├── 04-governance-defender.md
│   ├── 05-detection-engineering.md
│   ├── 06-red-team-findings.md
│   └── 07-infrastructure-as-code.md
├── bicep/
│   ├── main.bicep
│   ├── modules/
│   │   ├── identity.bicep
│   │   ├── network.bicep
│   │   ├── openai.bicep
│   │   ├── keyvault.bicep
│   │   ├── policy.bicep
│   │   └── sentinel.bicep
│   └── parameters/
│       └── main.parameters.json
├── kql/
│   ├── prompt-injection-detection.kql
│   ├── anomalous-token-consumption.kql
│   ├── off-hours-ai-access.kql
│   ├── impossible-travel-ai.kql
│   └── jailbreak-attempts.kql
├── policies/
│   ├── deny-public-ip.json
│   ├── require-private-endpoints.json
│   ├── require-managed-identity.json
│   └── require-diagnostic-settings.json
├── playbooks/
│   └── auto-disable-jailbreak-user.json
├── diagrams/
│   ├── architecture-overview.png
│   ├── network-topology.png
│   └── detection-pipeline.png
└── screenshots/
    ├── phase-01/
    ├── phase-02/
    ├── phase-03/
    ├── phase-04/
    ├── phase-05/
    ├── phase-06/
    └── phase-07/
```

---

## Security Frameworks Applied

- **Zero Trust Architecture** — NIST SP 800-207
- **NIST 800-53** — Security and Privacy Controls
- **NIST 800-61** — Computer Security Incident Handling
- **NIST AI Risk Management Framework** — AI RMF 1.0
- **CIS Microsoft Azure Foundations Benchmark**
- **Microsoft Cloud Adoption Framework (CAF)** — Security & Governance baseline
- **OWASP Top 10 for LLM Applications**

---

## Tools & Technologies

| Category | Technologies |
|---|---|
| **Identity** | Microsoft Entra ID P2, Conditional Access, PIM, Managed Identities |
| **Network** | Azure VNet, Private Endpoints, NSGs, Azure Bastion |
| **AI Workload** | Azure OpenAI Service, Content Filters, Prompt Shields |
| **Encryption** | Azure Key Vault, Customer-Managed Keys |
| **Governance** | Azure Policy, Regulatory Compliance, Initiatives |
| **Threat Protection** | Microsoft Defender for Cloud, Defender for AI Services |
| **SIEM/SOAR** | Microsoft Sentinel, Log Analytics, Logic Apps |
| **Query Language** | KQL (Kusto Query Language) |
| **Infrastructure as Code** | Bicep, ARM Templates |

---

## Key Learnings & Findings

*This section will be updated as each phase completes.*

---

<div align="center">

*Michael Edwards · Cybersecurity Engineer · Houston, TX*  
[GitHub](https://github.com/michaeledwards0) · [LinkedIn](https://linkedin.com/in/edwardsmichela)

</div>
