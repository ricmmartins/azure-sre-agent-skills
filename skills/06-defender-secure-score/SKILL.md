---
name: defender-secure-score-monitor
description: Monitor and improve Microsoft Defender for Cloud Secure Score. Tracks score trends, identifies quick wins to improve the score, checks Defender plan coverage, and highlights critical security recommendations. Use when asked about secure score, security posture, Defender status, or security improvements.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# Defender Secure Score Monitor

## Purpose
Track and improve your Microsoft Defender for Cloud Secure Score through focused monitoring, trend analysis, and actionable remediation guidance. Answers "what's our security score and how do we improve it?"

## When to use this skill
- User asks "what's our secure score?"
- User asks how to improve security posture
- Weekly security posture check
- Before a security audit or customer questionnaire
- After deploying new resources (to check score impact)

## Difference from Compliance/Governance Audit
The Compliance skill does a broad governance audit (policy, RBAC, tagging, locks, naming). This skill is **laser-focused on Defender for Cloud Secure Score** — the single most important security KPI — and provides targeted actions to improve it.

## Assessment procedure

### Step 1: Current Secure Score
Get the overall score and per-control breakdown.

```bash
az rest --method get --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.Security/secureScores/ascScore?api-version=2020-01-01"
```

```bash
az rest --method get --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.Security/secureScores/ascScore/secureScoreControls?api-version=2020-01-01"
```

Note: The `az security secure-scores list` CLI command may fail in some environments. The REST API approach above is more reliable. Replace `<sub-id>` with the target subscription ID.

Report:
- **Current Secure Score**: X% (X/Y points)
- **Trend** (if memory available): improved/declined vs. last check
- Top 5 controls with the most room for improvement (highest `max - current`)

### Step 2: Unhealthy assessments (recommendations)
Get the list of active security recommendations.

```bash
az rest --method get --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.Security/assessments?api-version=2021-06-01" --query "value[?properties.status.code=='Unhealthy'].{displayName:properties.displayName, severity:properties.metadata.severity, category:properties.metadata.categories[0], resourceId:properties.resourceDetails.id}"
```

Group by severity:
- 🔴 **High**: Must fix — significant risk
- 🟡 **Medium**: Should fix — moderate risk
- 🟢 **Low**: Nice to have — minimal risk

### Step 3: Quick wins (most points for least effort)
Identify recommendations that:
1. Affect the most resources (batch fix possible)
2. Have the highest point value per control
3. Can be fixed with a single CLI command or policy

Examples of quick wins:
- Enable HTTPS-only on App Services
- Enable secure transfer on storage accounts
- Enable Defender plans that are off
- Restrict public network access

### Step 4: Defender plan coverage
Check which Defender plans are enabled.

```bash
az security pricing list --query "[].{plan:name, tier:pricingTier, subPlan:subPlan}" -o table
```

Expected plans to be enabled for full coverage:
- Servers (P2 recommended)
- App Service
- SQL (databases)
- Storage
- Containers (with vulnerability assessment)
- Key Vault
- Resource Manager (ARM)
- DNS

Flag disabled plans and estimate score impact of enabling them.

### Step 5: Attack surface indicators
Check the most exploitable security gaps.

1. **Public management ports** (SSH/RDP open to internet):
   ```bash
   az graph query -q "resources | where type == 'microsoft.network/networksecuritygroups' | mvexpand rules = properties.securityRules | where rules.properties.access == 'Allow' and rules.properties.direction == 'Inbound' and (rules.properties.destinationPortRange == '22' or rules.properties.destinationPortRange == '3389' or rules.properties.destinationPortRange == '*') and rules.properties.sourceAddressPrefix in ('*', '0.0.0.0/0', 'Internet')" -o table
   ```

2. **Storage accounts with public blob access**:
   ```bash
   az storage account list --query "[?allowBlobPublicAccess==true].{name:name, rg:resourceGroup}" -o table
   ```

3. **SQL servers with public endpoint**:
   ```bash
   az sql server list --query "[?properties.publicNetworkAccess=='Enabled'].{name:name, rg:resourceGroup}" -o table
   ```

4. **Key Vaults accessible from public network**:
   ```bash
   az keyvault list --query "[?properties.publicNetworkAccess=='Enabled'].{name:name, rg:resourceGroup}" -o table
   ```

### Step 6: Credential and certificate hygiene
Check for expiring secrets that could cause outages or security drift.

1. **Key Vault secrets expiring within 30 days** (per vault):
   ```bash
   az keyvault secret list --vault-name <vault> --query "[?attributes.expires!=null && attributes.expires<'$(date -u -d '30 days' +%Y-%m-%dT%H:%M:%SZ)']" -o table
   ```

2. **Key Vault certificates expiring within 30 days** (per vault):
   ```bash
   az keyvault certificate list --vault-name <vault> --query "[?attributes.expires!=null && attributes.expires<'$(date -u -d '30 days' +%Y-%m-%dT%H:%M:%SZ)']" -o table
   ```

3. **App registrations with expiring secrets**:
   ```bash
   az ad app list --query "[].{name:displayName, creds:passwordCredentials[?endDateTime<'$(date -u -d '30 days' +%Y-%m-%dT%H:%M:%SZ)']}" --all -o table
   ```

Note on date handling: The SRE Agent sandbox runs Linux. Use `$(date -u -d '30 days' +%Y-%m-%dT%H:%M:%SZ)` for date calculations.

## Risk scoring

| Level | Criteria | Color |
|-------|----------|-------|
| **Critical** | Active exposure, immediate risk of breach | 🔴 |
| **High** | Significant gap, exploit requires minimal effort | 🟠 |
| **Medium** | Gap exists but requires specific conditions to exploit | 🟡 |
| **Low** | Best practice deviation, minimal immediate risk | 🟢 |

## Expected output

### Report header (mandatory — use this exact format)

## Defender Secure Score Report

| Field | Value |
|-------|-------|
| Subscription | (name + ID) |
| Assessment Date | YYYY-MM-DD |
| Overall Score | XX.XX / YY (ZZ%) |

Then present the gap summary:

> The subscription is earning only **ZZ%** of its maximum Secure Score. There is a **N-point gap** to close.

### Score improvement plan

| Priority | Action | Points | Effort | Resources affected |
|----------|--------|--------|--------|-------------------|
| 1 | Enable HTTPS-only on 4 App Services | +4 | 5 min | app-web-*, app-api-* |
| 2 | Enable Defender for Storage | +3 | 2 min | subscription-level |
| 3 | Restrict SQL public access on 2 servers | +3 | 30 min | sql-prod-*, sql-staging-* |
| 4 | Add NSG to 2 subnets without rules | +2 | 15 min | subnet-data-*, subnet-mgmt-* |
| **Total achievable** | | **+12** | **~1 hour** | |

### Critical findings (fix now)
Items that represent immediate exploitable risk.

### Remediation commands
Pre-formatted az cli commands for each quick win, ready to approve and execute.
Use `GetAzCliHelp` to validate the command syntax before suggesting.
```bash
# Enable HTTPS-only on App Service
az webapp update --name <app> --resource-group <rg> --set httpsOnly=true

# Enable Defender for Storage
az security pricing create --name StorageAccounts --tier Standard
```

### References
- Defender for Cloud: https://learn.microsoft.com/en-us/azure/defender-for-cloud/secure-score-security-controls
- Security Recommendations: https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference
- Defender Plans: https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction

### Recommended cadence
Schedule this skill as a **weekly** task via SRE Agent scheduled tasks for continuous score improvement tracking.
