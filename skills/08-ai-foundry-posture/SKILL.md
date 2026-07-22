---
name: ai-foundry-posture-check
description: >
  Assess security, reliability, and cost posture of Azure OpenAI and Microsoft Foundry 
  deployments. Detects critical anti-patterns: API keys instead of Managed Identity, 
  public endpoints without firewall, disabled content filtering, missing diagnostic 
  settings, deprecated model versions, over-provisioned PTU, absence of AI Gateway 
  (APIM) for rate limiting, shared dev/prod deployments, quota exhaustion risk, 
  429 throttling rates, runaway token consumption, and single-region deployment. 
  Use when asked about AI workload security, OpenAI best practices, Foundry health 
  check, model deployment review, AI cost optimization, quota monitoring, or 
  throttling analysis.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# AI Foundry & OpenAI Posture Check

## Purpose
Assess the security, reliability, and cost efficiency of Azure OpenAI and Microsoft Foundry deployments. Detects the most common anti-patterns that startups make when building AI-powered products — from exposed endpoints to runaway token costs.

Based on the Azure Well-Architected Framework for AI workloads and Microsoft Foundry operational best practices.

## When to use this skill
- User asks "is our OpenAI deployment secure?"
- User asks about AI cost optimization or token consumption
- Review before going to production with an AI feature
- User asks about content filtering, model versions, or rate limiting
- Periodic AI workload health check

## Pre-check
Confirm with the user:
- Which Azure OpenAI / Cognitive Services accounts to assess (or "all in subscription")
- Whether they use PTU (Provisioned Throughput) or Standard deployments
- Whether they have production AI workloads already live

## Assessment procedure

### Step 0: Discover AI resources

```bash
az cognitiveservices account list \
  --subscription <sub-id> \
  --query "[?kind=='OpenAI' || kind=='AIServices'].{name:name, kind:kind, rg:resourceGroup, location:location, sku:sku.name}" \
  -o table
```

If no results, try:
```bash
az cognitiveservices account list \
  --subscription <sub-id> \
  --query "[].{name:name, kind:kind, rg:resourceGroup, location:location}" \
  -o table
```

If no Cognitive Services accounts exist, report "No Azure OpenAI or AI Foundry resources found" and end assessment.

For each account found, run the following checks:

---

### 🔐 CATEGORY 1 — Security (Critical)

**Check 1.1 — Managed Identity enabled (not API keys only)**

```bash
az cognitiveservices account show \
  --name <account> --resource-group <rg> \
  --query "{identity:identity.type, disableLocalAuth:properties.disableLocalAuth}" \
  -o json
```

| Finding | Severity | Score |
|---------|----------|-------|
| `identity.type` = SystemAssigned/UserAssigned AND `disableLocalAuth` = true | ✅ Pass | 12 pts |
| `identity.type` set but `disableLocalAuth` = false | ⚠️ Partial | 6 pts |
| `identity.type` = None or null | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/managed-identity

**Check 1.2 — Network isolation (firewall or private endpoint)**

```bash
az cognitiveservices account show \
  --name <account> --resource-group <rg> \
  --query "{publicAccess:properties.publicNetworkAccess, defaultAction:properties.networkAcls.defaultAction, privateEndpoints:properties.privateEndpointConnections}" \
  -o json
```

| Finding | Severity | Score |
|---------|----------|-------|
| `publicNetworkAccess` = Disabled + private endpoints exist | ✅ Pass | 13 pts |
| `defaultAction` = Deny + IP/VNet rules configured | ✅ Pass | 13 pts |
| `defaultAction` = Allow (open to internet) | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/cognitive-services-virtual-networks

**Check 1.3 — Content filtering (RAI policies)**

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/raiPolicies?api-version=2024-10-01"
```

Then check deployments have a policy assigned:
```bash
az cognitiveservices account deployment list \
  --name <account> --resource-group <rg> \
  --query "[].{name:name, model:properties.model.name, raiPolicy:properties.raiPolicyName}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| All deployments have RAI policy assigned with filters enabled | ✅ Pass | 10 pts |
| Some deployments missing policy or using minimal filters | ⚠️ Partial | 5 pts |
| No RAI policies or content filtering disabled | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/content-filters

### ⚙️ CATEGORY 2 — Reliability & Operations

**Check 2.1 — Model version pinned and not deprecated**

```bash
az cognitiveservices account deployment list \
  --name <account> --resource-group <rg> \
  --query "[].{name:name, model:properties.model.name, version:properties.model.version}" \
  -o table
```

Cross-reference with available models:
```bash
az cognitiveservices account list-models \
  --name <account> --resource-group <rg> \
  --query "[].{model:model.name, version:model.version, lifecycle:model.lifecycleStatus}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| All deployments on GA versions with explicit version pinned | ✅ Pass | 7 pts |
| Deployments on versions nearing retirement (< 3 months) | ⚠️ Attention | 3 pts |
| Deployments on Legacy/Deprecated versions | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/model-retirements

**Check 2.2 — Diagnostic settings enabled**

```bash
# Get resource ID
RESOURCE_ID=$(az cognitiveservices account show \
  --name <account> --resource-group <rg> --query "id" -o tsv)

az monitor diagnostic-settings list --resource $RESOURCE_ID -o json
```

| Finding | Severity | Score |
|---------|----------|-------|
| Diagnostic settings sending RequestResponse + Audit to Log Analytics | ✅ Pass | 7 pts |
| Diagnostic settings exist but incomplete (only metrics, no logs) | ⚠️ Partial | 3 pts |
| No diagnostic settings at all | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/monitoring

**Check 2.3 — Resource Lock on production account**

```bash
az lock list --resource-group <rg> \
  --query "[?contains(id, '<account>')].{name:name, level:level}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| CanNotDelete lock exists on the OpenAI account | ✅ Pass | 5 pts |
| Lock exists on RG but not specifically on account | ⚠️ Partial | 3 pts |
| No locks | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources

**Check 2.4 — Multi-region resilience (failover capability)**

Check if OpenAI accounts exist in more than one region (disaster recovery / latency optimization):

```bash
az cognitiveservices account list \
  --subscription <sub-id> \
  --query "[?kind=='OpenAI' || kind=='AIServices'].{name:name, location:location, rg:resourceGroup}" \
  -o table
```

Group by location and count unique regions:

| Finding | Severity | Score |
|---------|----------|-------|
| OpenAI accounts in 2+ different regions | ✅ Pass | 5 pts |
| All OpenAI accounts in a single region | ⚠️ Risk | 2 pts |
| Only 1 OpenAI account total (single point of failure) | ❌ Risk | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/business-continuity-disaster-recovery

> **Context from the field:** Customers with single-region deployments face outages during regional capacity events. Multi-region + APIM load balancing (Check 4.1) is the recommended pattern for production AI workloads.

**Check 2.5 — 429 throttling rate (last 24 hours)**

Requires diagnostic settings sending logs to Log Analytics (Check 2.2). If diagnostics are not configured, skip and note dependency.

```kusto
AzureDiagnostics
| where TimeGenerated > ago(24h)
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| summarize 
    TotalRequests = count(),
    ThrottledRequests = countif(resultSignature_d == 429),
    ThrottleRate = round(100.0 * countif(resultSignature_d == 429) / count(), 2)
  by Resource, properties_s
| order by ThrottleRate desc
```

Use `execute_kusto_query` with the Log Analytics workspace connected to the OpenAI account's diagnostic settings.

| Finding | Severity | Score |
|---------|----------|-------|
| Throttle rate < 1% in last 24h | ✅ Pass | 6 pts |
| Throttle rate 1–10% (occasional bursts) | ⚠️ Attention | 3 pts |
| Throttle rate > 10% (active capacity problem) | ❌ Fail | 0 pts |
| No diagnostic data available (Check 2.2 failed) | ⚠️ Skip | 0 pts — flag dependency |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota

> **Context from the field:** Customers assume PTU eliminates throttling — it does not. Bursty traffic and concurrency limits can cause 429s even on PTU. Check APIM retry/spillover pattern (Check 4.1) as mitigation.

---

### 💰 CATEGORY 3 — Cost & Efficiency

**Check 3.1 — Rate limits configured per deployment (not max)**

```bash
az cognitiveservices account deployment list \
  --name <account> --resource-group <rg> \
  --query "[].{name:name, model:properties.model.name, sku:sku.name, capacity:sku.capacity}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| Each deployment has explicit TPM capacity set (not maximum) | ✅ Pass | 5 pts |
| Single deployment consuming all available quota | ⚠️ Risk | 2 pts |
| Unable to determine (no deployments) | N/A | 5 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota

**Check 3.2 — Model selection diversity (not GPT-4o for everything)**

```bash
az cognitiveservices account deployment list \
  --name <account> --resource-group <rg> \
  --query "[].{name:name, model:properties.model.name}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| Mix of models (e.g., gpt-4o + gpt-4o-mini, or batch deployments) | ✅ Pass | 3 pts |
| Only premium models deployed (no cost-efficient alternative) | ⚠️ Info | 1 pt |
| Single deployment only | N/A | 3 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models

**Check 3.3 — PTU utilization (if applicable)**

Only check if PTU/Provisioned deployments exist:
```bash
az cognitiveservices account deployment list \
  --name <account> --resource-group <rg> \
  --query "[?sku.name=='ProvisionedManaged'].{name:name, capacity:sku.capacity}" \
  -o table
```

If PTU deployments exist, check utilization via metrics:
```bash
az rest --method get \
  --url "https://management.azure.com<resource-id>/providers/microsoft.insights/metrics?api-version=2019-07-01&metricnames=ProvisionedManagedUtilizationV2&timespan=P7D&interval=PT1H&aggregation=Average"
```

| Finding | Severity | Score |
|---------|----------|-------|
| PTU utilization 70–85% average (well-sized) | ✅ Pass | 7 pts |
| PTU utilization < 50% (over-provisioned — burning money) | ⚠️ Waste | 2 pts |
| PTU utilization > 95% (under-provisioned — users getting 429s) | ⚠️ Risk | 4 pts |
| No PTU deployments (using Standard — fine for most startups) | N/A | 7 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/provisioned-throughput

**Check 3.4 — Quota utilization (TPM used vs limit)**

```bash
az cognitiveservices account list-usage \
  --name <account> --resource-group <rg> \
  -o json
```

Alternative if the above returns empty (common for newer deployments):
```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.CognitiveServices/locations/<location>/usages?api-version=2024-10-01"
```

| Finding | Severity | Score |
|---------|----------|-------|
| Quota usage < 70% of limit across all models | ✅ Pass | 5 pts |
| Quota usage 70–90% (nearing limit — plan increase) | ⚠️ Attention | 3 pts |
| Quota usage > 90% (at risk of rejection — request increase NOW) | ❌ Risk | 0 pts |
| Unable to retrieve usage data | ⚠️ Info | 3 pts |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits

> **Context from the field:** Approved TPM quotas sometimes don't appear in Foundry due to subscription/project mismatches or the new Global/Data Zone quota pooling model. If usage data looks incorrect, verify the quota was assigned to the correct subscription and region.

**Check 3.5 — Token consumption trend (tokens per day per deployment)**

Requires diagnostic settings sending logs to Log Analytics (Check 2.2). If diagnostics are not configured, skip and note dependency.

```kusto
AzureDiagnostics
| where TimeGenerated > ago(7d)
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| extend model = tostring(parse_json(properties_s).modelName)
| extend promptTokens = toint(parse_json(properties_s).promptTokens)
| extend completionTokens = toint(parse_json(properties_s).completionTokens)
| summarize 
    DailyPromptTokens = sum(promptTokens),
    DailyCompletionTokens = sum(completionTokens),
    DailyTotalTokens = sum(promptTokens) + sum(completionTokens),
    RequestCount = count()
  by bin(TimeGenerated, 1d), Resource, model
| order by TimeGenerated desc, DailyTotalTokens desc
```

Use `execute_kusto_query` with the Log Analytics workspace.

| Finding | Severity | Score |
|---------|----------|-------|
| Token consumption stable or declining (no runaway growth) | ✅ Pass | 5 pts |
| Token consumption growing > 20% day-over-day (investigate) | ⚠️ Attention | 2 pts |
| Token consumption spiking > 50% day-over-day (possible prompt leak or loop) | ❌ Risk | 0 pts |
| No diagnostic data available (Check 2.2 failed) | ⚠️ Skip | 0 pts — flag dependency |

> 📖 https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/monitoring

> **Context from the field:** Unmonitored token growth is the #1 surprise cost driver for startups using OpenAI. A single prompt injection loop or chatbot without max_tokens can burn through thousands of dollars in a weekend.

---

### 🏗️ CATEGORY 4 — Architecture Patterns

**Check 4.1 — AI Gateway / APIM in front of OpenAI (for production)**

```bash
az apim list --subscription <sub-id> \
  --query "[].{name:name, sku:sku.name, rg:resourceGroup}" \
  -o table
```

| Finding | Severity | Score |
|---------|----------|-------|
| APIM exists with OpenAI-related APIs configured | ✅ Pass | 5 pts |
| No APIM — calls go directly to OpenAI endpoint | ⚠️ Risk | 0 pts |
| N/A (dev/test environment only) | N/A | 5 pts |

> 📖 https://learn.microsoft.com/en-us/azure/api-management/api-management-authenticate-authorize-azure-openai

**Check 4.2 — Environment separation (dev ≠ prod for AI resources)**

```bash
az cognitiveservices account list \
  --subscription <sub-id> \
  --query "[?kind=='OpenAI'].{name:name, rg:resourceGroup, tags:tags}" \
  -o json
```

| Finding | Severity | Score |
|---------|----------|-------|
| Separate accounts or deployments for dev/prod with tags | ✅ Pass | 5 pts |
| Single account with environment tag but shared deployments | ⚠️ Partial | 2 pts |
| Single account, no separation, no tags | ❌ Fail | 0 pts |

> 📖 https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions

## Scoring

| Category | Checks | Max Points |
|----------|--------|-----------|
| 🔐 Security (Identity, Network, Content Filter) | 1.1–1.3 | 35 |
| ⚙️ Reliability (Model version, Diagnostics, Locks, Multi-region, 429 monitoring) | 2.1–2.5 | 30 |
| 💰 Cost & Efficiency (Rate limits, Model mix, PTU, Quota, Token trend) | 3.1–3.5 | 25 |
| 🏗️ Architecture (Gateway, Env separation) | 4.1–4.2 | 10 |
| **TOTAL** | **15 checks** | **100** |

### Maturity levels

| Score | Level | Meaning |
|-------|-------|---------|
| 0–39 | 🔴 **Critical** | Major security and operational gaps — not production-ready |
| 40–69 | 🟡 **Developing** | Some controls in place but significant risks remain |
| 70–89 | 🟢 **Solid** | Good posture — minor improvements recommended |
| 90–100 | 🏆 **Exemplary** | Following WAF best practices for AI workloads |

## Expected output

### Header
```
╔══════════════════════════════════════════════════════════════════╗
║       AI FOUNDRY & OPENAI POSTURE CHECK                         ║
║                                                                  ║
║   Score: XX / 100         Level: 🟡 Developing                  ║
║   Accounts assessed: N                                           ║
║   Deployments found: M                                           ║
╚══════════════════════════════════════════════════════════════════╝
```

### Per-account findings

For each OpenAI/AI Services account:

| Check | Status | Finding | Recommendation |
|-------|--------|---------|----------------|
| Managed Identity | ❌ | API keys enabled, no MI | Enable System MI + set disableLocalAuth=true |
| Network | ⚠️ | Public access enabled, no firewall | Set defaultAction=Deny + add VNet rules |
| ... | ... | ... | ... |

### Critical findings (fix immediately)
Security issues that expose the AI endpoint or violate responsible AI principles.

### Cost optimization opportunities
Token waste, PTU over-provisioning, or model selection improvements with estimated savings.

### Architecture recommendations
Patterns to improve resilience: AI Gateway, retry policies, multi-region deployment.

### Remediation guidance
For each ❌ or ⚠️ finding, include in the output:
1. The specific `az` CLI command to remediate (suggest only — do not execute)
2. Use `GetAzCliHelp` to validate the command syntax before suggesting
3. Include the 📖 documentation link from the check that failed

### References
- WAF for AI: https://learn.microsoft.com/en-us/azure/well-architected/ai/
- OpenAI Security: https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/managed-identity
- Content Filtering: https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/content-filters
- Model Lifecycle: https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/model-retirements
- PTU Planning: https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/provisioned-throughput
- PTU Calculator: https://ptucalc.com
