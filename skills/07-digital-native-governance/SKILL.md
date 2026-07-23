---
name: digital-native-governance-check
description: >
  Assess Azure governance maturity for Digital Natives and startups. Detects the most common 
  anti-patterns from the "Digital Natives on Azure" series: incomplete Service Health alerts, 
  flat subscription topology without Management Groups, excessive RBAC (everyone is Owner), 
  missing budget alerts, no Managed Identity usage, unprotected web apps without WAF/DDoS, 
  and   missing environment separation. Produces a scored maturity report 
  (Foundation → Developing → Established → Optimized). 
  Use when asked about startup readiness, governance maturity, "are we production-ready?", 
  or before a customer security questionnaire.
tools:
  - RunAzCliReadCommands
---

# Digital Native Governance Check

## Purpose
Evaluate whether a startup's Azure environment has the foundational governance that enterprise customers, investors, and auditors expect. Produces a maturity score (0–100) with clear next steps.

This is NOT a deep compliance audit (use the Compliance & Governance skill for that). This is a focused "did you get the basics right?" diagnostic for teams of 5–50 engineers who set up Azure fast and moved on.

## When to use this skill
- Startup preparing for first enterprise customer security review
- CTO asks "are we production-ready?"
- Pre-Series A infrastructure maturity check
- After first incident: "what basics did we miss?"
- Quarterly governance health check for Digital Natives

## Maturity levels

| Score | Level | Meaning |
|-------|-------|---------|
| 0–39 | 🔴 **Foundation** | Governance basics missing — will fail any security questionnaire |
| 40–69 | 🟡 **Developing** | Some controls exist but significant gaps remain |
| 70–89 | 🟢 **Established** | Solid posture — can pass most security reviews |
| 90–100 | 🏆 **Optimized** | Enterprise-grade governance on Azure |

## Assessment procedure

Run all checks in parallel where possible. Score each and sum for total.

---

### 🏥 CATEGORY 1 — Alerting & Observability (20 points)

Detects: "Azure VM went down and nobody knew" + "you configured one alert type out of four"

**Check 1.1 — Service Health alerts exist (8 pts)**

```bash
az monitor activity-log alert list \
  --subscription <sub-id> \
  --query "[?contains(to_string(condition.allOf), 'ServiceHealth')].{name:name, enabled:enabled}" \
  -o json
```

Scoring:
- 8 pts: Alert exists covering all 4 event types (Service Issues, Health Advisories, Planned Maintenance, Security Advisories)
- 4 pts: Alert exists but covers only 1–2 types
- 0 pts: No Service Health alerts at all

> 📖 https://learn.microsoft.com/en-us/azure/service-health/alerts-activity-log-service-notifications-portal

**Check 1.2 — Resource Health alerts for compute (6 pts)**

```bash
az monitor activity-log alert list \
  --subscription <sub-id> \
  --query "[?contains(to_string(condition.allOf), 'ResourceHealth')].{name:name, enabled:enabled}" \
  -o json
```

Scoring:
- 6 pts: Resource Health alerts exist and are enabled
- 0 pts: No Resource Health alerts — VMs can go Unavailable silently

> 📖 https://learn.microsoft.com/en-us/azure/service-health/resource-health-alert-monitor-guide

**Check 1.3 — Action Groups with valid receivers (6 pts)**

```bash
az monitor action-group list \
  --subscription <sub-id> \
  --query "[].{name:name, emails:length(emailReceivers), webhooks:length(webhookReceivers)}" \
  -o table
```

Scoring:
- 6 pts: Action group with email + at least one other channel (webhook, SMS)
- 3 pts: Action group with email only
- 0 pts: No action groups configured

> 📖 https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups

---

### 🏗️ CATEGORY 2 — Subscription Topology (20 points)

Detects: "the flat subscription problem" + "dev/test/prod in the same subscription"

**Check 2.1 — Management Groups beyond Tenant Root (8 pts)**

```bash
az account management-group list \
  --query "[].{name:name, displayName:displayName}" \
  -o table
```

Scoring:
- 8 pts: Custom MG hierarchy exists (nonprod/prod/platform pattern)
- 4 pts: At least one custom MG beyond Tenant Root Group
- 0 pts: Only Tenant Root Group — completely flat

> 📖 https://learn.microsoft.com/en-us/azure/governance/management-groups/overview

**Check 2.2 — Environment separation (8 pts)**

```bash
az account list --all \
  --query "[?state=='Enabled'].{name:name, id:id}" \
  -o table
```

Look for prod/nonprod/dev/staging patterns in subscription names.

Scoring:
- 8 pts: Clear separation (prod ≠ nonprod in distinct subscriptions)
- 4 pts: Multiple subscriptions but unclear separation
- 0 pts: Single subscription for everything

> 📖 https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions

**Check 2.3 — Resource Locks on critical infrastructure (4 pts)**

```bash
az lock list --subscription <sub-id> \
  --query "[].{name:name, level:level, notes:notes}" \
  -o table
```

Scoring:
- 4 pts: CanNotDelete locks on infra resources (Log Analytics, Key Vault, Terraform state)
- 2 pts: Locks exist but only on non-critical resources
- 0 pts: Zero locks — anything can be accidentally deleted

> 📖 https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources

---

### 💰 CATEGORY 3 — Cost Controls (15 points)

Detects: "surprise bill" + "VMs running in dev all weekend"

**Check 3.1 — Budget alerts configured (8 pts)**

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.Consumption/budgets?api-version=2023-11-01"
```

Scoring:
- 8 pts: Budget exists with notifications at 80% and 100%
- 4 pts: Budget exists but missing thresholds or notifications
- 0 pts: No budgets — first warning is the invoice

> 📖 https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets

**Check 3.2 — Azure Advisor cost recommendations addressed (4 pts)**

```bash
az advisor recommendation list --category Cost \
  --query "[?impact=='High'].{problem:shortDescription.problem}" \
  -o table
```

Scoring:
- 4 pts: Zero high-impact cost recommendations pending
- 2 pts: 1–3 high-impact recommendations
- 0 pts: 4+ unaddressed — systematic waste

> 📖 https://learn.microsoft.com/en-us/azure/advisor/advisor-cost-recommendations

**Check 3.3 — No cost-leaking VMs (3 pts)**

```bash
az vm list -d --subscription <sub-id> \
  --query "[?powerState=='VM stopped'].{name:name, rg:resourceGroup}" \
  -o table
```

Note: "VM stopped" (not deallocated) still incurs compute charges.

Scoring:
- 3 pts: No VMs in stopped-but-allocated state
- 0 pts: VMs burning money while stopped

> 📖 https://learn.microsoft.com/en-us/azure/virtual-machines/states-billing

---

### 🔐 CATEGORY 4 — Identity & Access (20 points)

Detects: "everyone is Owner" + "secrets in code instead of Managed Identity"

**Check 4.1 — Owner count ≤ 3 per subscription (6 pts)**

```bash
az role assignment list --role Owner \
  --subscription <sub-id> \
  --query "[].{principal:principalName, type:principalType}" \
  -o table
```

Scoring:
- 6 pts: ≤ 3 Owners at subscription level
- 3 pts: 4–5 Owners
- 0 pts: 6+ Owners — the "everyone is Owner for convenience" anti-pattern

> 📖 https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices

**Check 4.2 — Managed Identities in use (7 pts)**

```bash
az identity list --subscription <sub-id> \
  --query "[].{name:name, rg:resourceGroup}" \
  -o table
```

```bash
az vm list --subscription <sub-id> \
  --query "[].{name:name, identityType:identity.type}" \
  -o table
```

Scoring:
- 7 pts: User-Assigned or System-Assigned MI actively used on resources
- 3 pts: Some resources have MI, key resources don't
- 0 pts: No Managed Identities — likely using keys/secrets everywhere

> 📖 https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview

**Check 4.3 — Defender for Cloud enabled (7 pts)**

```bash
az security pricing list \
  --query "value[?name=='CloudPosture' || name=='VirtualMachines' || name=='StorageAccounts'].{service:name, tier:pricingTier}" \
  -o table
```

Scoring:
- 7 pts: CSPM + at least one workload protection (Standard tier)
- 4 pts: Only free tier CSPM
- 0 pts: Defender not configured — zero security visibility

> 📖 https://learn.microsoft.com/en-us/azure/defender-for-cloud/get-started

---

### 🌐 CATEGORY 5 — Production Web Protection (15 points)

Detects: "startup running prod on .azurewebsites.net with no WAF"

**Check 5.1 — Custom domain on production web apps (5 pts)**

```bash
az webapp list --subscription <sub-id> \
  --query "[].{name:name, defaultHost:defaultHostName, customDomains:hostNames}" \
  -o json
```

```bash
az staticwebapp list --subscription <sub-id> \
  --query "[].{name:name, defaultHost:defaultHostname, customDomains:customDomains}" \
  -o json
```

Scoring:
- 5 pts: Production apps use custom domains
- 2 pts: Mix of custom and default domains
- 0 pts: Production on `.azurewebsites.net` / `.azurestaticapps.net`
- N/A: No web apps (score 5 pts — not applicable)

> 📖 https://learn.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-custom-domain

**Check 5.2 — WAF or Front Door protecting web endpoints (5 pts)**

```bash
az network application-gateway waf-policy list \
  --subscription <sub-id> \
  --query "[].{name:name, rg:resourceGroup}" \
  -o table
```

```bash
az afd profile list --subscription <sub-id> \
  --query "[].{name:name, sku:sku.name}" \
  -o table
```

Scoring:
- 5 pts: WAF policy active OR Front Door Premium/Standard protecting endpoints
- 2 pts: Front Door exists but without WAF enabled
- 0 pts: Web apps directly exposed without any L7 protection
- N/A: No public web apps (score 5 pts)

> 📖 https://learn.microsoft.com/en-us/azure/web-application-firewall/overview

**Check 5.3 — DDoS protection stance (5 pts)**

```bash
az network ddos-protection list \
  --subscription <sub-id> \
  --query "[].{name:name, rg:resourceGroup}" \
  -o table
```

Scoring:
- 5 pts: DDoS plan active on VNets with public IPs, OR architecture is PaaS-only (built-in protection)
- 0 pts: VMs/IPs exposed without DDoS plan

Note: Pure PaaS architectures (App Service, Static Web Apps, Container Apps without VNet) get Azure's built-in DDoS at the platform level. Score 5 if no VNets with public IPs exist.

> 📖 https://learn.microsoft.com/en-us/azure/ddos-protection/ddos-protection-overview

---

### 📋 CATEGORY 6 — Operational Foundations (10 points)

**Check 6.1 — Activity Log forwarding to Log Analytics (5 pts)**

```bash
az monitor diagnostic-settings subscription list \
  --subscription <sub-id> \
  -o json
```

Scoring:
- 5 pts: Activity Logs forwarded to Log Analytics workspace
- 2 pts: Forwarded to storage only (limited query capability)
- 0 pts: No diagnostic settings — no audit trail

> 📖 https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log

**Check 6.2 — Backup exists for stateful resources (5 pts)**

```bash
az backup vault list --subscription <sub-id> \
  --query "[].{name:name, rg:resourceGroup}" \
  -o table
```

Scoring:
- 5 pts: Recovery Services vault with active backup items
- 2 pts: Vault exists but empty (no backup items)
- 0 pts: No backup — one deletion away from data loss

> 📖 https://learn.microsoft.com/en-us/azure/backup/backup-overview

---

## Expected output

### Report header (mandatory — use this exact format)

## Digital Native Governance Check Report

| Field | Value |
|-------|-------|
| Score | XX / 100 |
| Level | 🟡 Developing |
| Subscription | (name + ID) |
| Assessment Date | YYYY-MM-DD |

### Score breakdown

| Category | Score | Max | Status |
|----------|-------|-----|--------|
| 🏥 Alerting & Observability | X | 20 | 🟢/🟡/🔴 |
| 🏗️ Subscription Topology | X | 20 | 🟢/🟡/🔴 |
| 💰 Cost Controls | X | 15 | 🟢/🟡/🔴 |
| 🔐 Identity & Access | X | 20 | 🟢/🟡/🔴 |
| 🌐 Web Protection | X | 15 | 🟢/🟡/🔴 |
| 📋 Operational Foundations | X | 10 | 🟢/🟡/🔴 |
| **TOTAL** | **XX** | **100** | **Level** |

Category status: 🟢 ≥ 80% | 🟡 40–79% | 🔴 < 40%

### 🚨 Critical gaps (blocks enterprise deals)
Items that would immediately fail a security questionnaire. List with severity.

### ⚡ Quick wins (fix this week)
Top 3 highest point-gain items with lowest effort. Include the `az` command to fix.

### 📈 Roadmap to next level
What to tackle next month to reach the next governance maturity level.

### Remediation guidance
For each ❌ or ⚠️ finding, include in the output:
1. The specific `az` CLI command to remediate (suggest only — do not execute)
2. Use `GetAzCliHelp` to validate the command syntax before suggesting
3. Include the 📖 documentation link from the check that failed

### 📚 Learn more
Link each finding to the relevant "Digital Natives on Azure" article for deep context.
