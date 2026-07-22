---
name: well-architected-review
description: Run a Well-Architected Framework (WAF) review against Azure resources in scope. Use when asked about best practices, architecture review, WAF assessment, or pillar compliance (Reliability, Security, Cost, Operational Excellence, Performance Efficiency).
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# Well-Architected Review

## Purpose
Perform a structured assessment of Azure resources against the five pillars of the Microsoft Azure Well-Architected Framework. Produce a scored report with prioritized recommendations.

## When to use this skill
- User asks "are we following best practices?"
- User asks for a WAF or Well-Architected review
- User wants to assess architecture quality before a launch or audit
- Periodic (quarterly) architecture health check

## Pillars and checks

### 1. Reliability
Run the following checks and report findings:

1. **Availability design**: Check if critical workloads use availability zones or availability sets
   ```bash
   az vm list --query "[].{name:name, zones:zones, availabilitySet:availabilitySet.id}" -o table
   az appservice plan list --query "[].{name:name, zoneRedundant:zoneRedundant, sku:sku.name}" -o table
   ```
2. **Backup coverage**: Verify Recovery Services vaults and backup policies exist for VMs, databases, and file shares
   ```bash
   az backup vault list -o table
   ```
   Then for each vault:
   ```bash
   az backup item list --vault-name <vault> --resource-group <rg> --backup-management-type AzureIaasVM -o table
   ```
3. **Disaster recovery**: Check for paired regions, ASR replication, or geo-redundant storage
4. **Health probes**: Verify App Service health checks, load balancer probes, and Container Apps health endpoints
5. **Auto-healing**: Check if App Service auto-heal rules or AKS pod disruption budgets are configured

### 2. Security
1. **Identity**: Check for managed identities vs. stored credentials
   ```bash
   az webapp identity show --name <app> --resource-group <rg>
   az ad app list --query "[].{name:displayName, passwordCredentials:passwordCredentials}" -o table
   ```
2. **Network isolation**: Check for private endpoints, NSGs, and service endpoints
   ```bash
   az network private-endpoint list -o table
   az network nsg list -o table
   ```
3. **Encryption**: Verify encryption at rest (storage, databases) and in transit (TLS)
4. **Key management**: Check Key Vault usage and key/secret expiration dates
   ```bash
   az keyvault list -o table
   az keyvault secret list --vault-name <vault> --query "[].{name:name, expires:attributes.expires}" -o table
   ```
5. **Defender for Cloud**: Check Secure Score and outstanding recommendations

### 3. Cost Optimization
1. **Rightsizing**: Identify underutilized VMs (CPU < 5% average over 14 days)
2. **Orphaned resources**: Find unattached disks, unused public IPs, empty resource groups
   ```bash
   az disk list --query "[?managedBy==null].{name:name, size:diskSizeGb, rg:resourceGroup}" -o table
   az network public-ip list --query "[?ipConfiguration==null].{name:name, rg:resourceGroup}" -o table
   ```
3. **Reservations**: Check if high-usage resources could benefit from reserved instances
4. **Dev/Test pricing**: Verify non-production workloads use Dev/Test subscriptions or B-series VMs
5. **Storage tiers**: Check if cool/archive tiers are used for infrequently accessed data

### 4. Operational Excellence
1. **IaC coverage**: Check for ARM/Bicep/Terraform templates in connected repos
2. **Tagging**: Verify mandatory tags (environment, owner, cost-center) exist
   ```bash
   az resource list --query "[?tags.environment==null].{name:name, type:type, rg:resourceGroup}" -o table
   ```
3. **Monitoring**: Check for alert rules, action groups, and diagnostic settings
   ```bash
   az monitor metrics alert list -o table
   az monitor diagnostic-settings list --resource <id> -o table
   ```
4. **Deployment practices**: Check for deployment slots, blue-green, or canary configurations
5. **Automation**: Check for runbooks, Logic Apps, or scheduled tasks for routine operations

### 5. Performance Efficiency
1. **Autoscaling**: Verify autoscale rules exist for App Service plans, VMSS, and Container Apps
   ```bash
   az monitor autoscale list --resource-group <rg> -o table
   ```
   Note: `az monitor autoscale list` requires `--resource-group`. Iterate over relevant resource groups, or use Azure Resource Graph:
   ```bash
   az graph query -q "resources | where type == 'microsoft.insights/autoscalesettings'" -o table
   ```
2. **Caching**: Check for Redis Cache or CDN usage on high-traffic workloads
3. **Database performance**: Check DTU/vCore utilization, index recommendations
4. **Content delivery**: Verify static assets use CDN or Front Door
5. **Connection pooling**: Check for connection string patterns suggesting missing pooling

## Scoring model

For each check, assign one of:
- ✅ **Pass** — follows best practice
- ⚠️ **Needs attention** — partially implemented or at risk
- ❌ **Fail** — not implemented, risk exposure

## Expected output

### Summary
A table with pillar scores:

| Pillar | Pass | Needs Attention | Fail | Score |
|--------|------|----------------|------|-------|
| Reliability | X | Y | Z | X/(X+Y+Z) % |
| Security | ... | ... | ... | ... |
| Cost Optimization | ... | ... | ... | ... |
| Operational Excellence | ... | ... | ... | ... |
| Performance Efficiency | ... | ... | ... | ... |
| **Overall** | | | | **avg %** |

### Detailed findings
For each pillar, list every check with:
- Status (pass/attention/fail)
- Evidence (command output or observation)
- Recommendation (specific action to take)
- Priority (Critical / High / Medium / Low)
- Reference link to WAF documentation

### Top 5 recommendations
Ordered by impact, with estimated effort (hours/days) for each.

### Remediation guidance
For each ❌ or ⚠️ finding, include in the output:
1. The specific `az` CLI command to remediate (suggest only — do not execute)
2. Use `GetAzCliHelp` to validate the command syntax before suggesting
3. The official Microsoft Learn documentation link for the remediation

### References
- WAF Overview: https://learn.microsoft.com/en-us/azure/well-architected/
- Reliability: https://learn.microsoft.com/en-us/azure/well-architected/reliability/
- Security: https://learn.microsoft.com/en-us/azure/well-architected/security/
- Cost Optimization: https://learn.microsoft.com/en-us/azure/well-architected/cost-optimization/
- Operational Excellence: https://learn.microsoft.com/en-us/azure/well-architected/operational-excellence/
- Performance Efficiency: https://learn.microsoft.com/en-us/azure/well-architected/performance-efficiency/
