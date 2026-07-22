---
name: capacity-planning
description: Assess Azure resource capacity, quota utilization, and growth trends to prevent outages from resource exhaustion. Use when asked about capacity, quotas, scaling readiness, load testing prep, Black Friday readiness, or growth projections.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# Capacity Planning

## Purpose
Analyze current resource utilization, quota consumption, and growth trends to predict capacity risks and recommend scaling actions before limits are hit.

## When to use this skill
- User asks "will our infrastructure handle the traffic spike?"
- User asks about quotas, limits, or capacity
- Pre-event planning (Black Friday, product launch, marketing campaign)
- Quarterly capacity review
- After hitting a quota or throttling limit

## Pre-check
Confirm with the user:
- Scope: which subscriptions and regions
- Time horizon: when is the expected load increase? (default: 30 days)
- Expected growth factor (e.g., 2x, 5x, 10x current load)
- Critical workloads to prioritize

## Assessment procedure

### Step 1: Quota utilization
Check current quota usage across all critical resource providers.

```bash
az vm list-usage --location <region> -o table
az network list-usages --location <region> -o table
az storage account list --query "length(@)"
```

Check for:
- **vCPU quotas**: Total and per-family (Dv5, Ev5, etc.) — flag if > 70% utilized
- **Public IPs**: Usage vs. limit
- **Load balancers**: Usage vs. limit
- **Network interfaces**: Usage vs. limit
- **Storage accounts per subscription**: Usage vs. 250 limit
- **App Service plans per region**
- **AKS clusters per subscription**

Build a quota utilization table. Use Unicode emoji characters directly (🟢🟡🔴) — never use emoji shortcodes like `:green_circle:` or `:red_circle:`.

| Resource | Region | Used | Limit | Utilization | Risk |
|----------|--------|------|-------|-------------|------|
| Total vCPUs | East US | 85 | 100 | 85% | 🔴 |
| Dv5 vCPUs | East US | 32 | 50 | 64% | 🟡 |
| Public IPs | East US | 8 | 20 | 40% | 🟢 |

### Step 2: Compute capacity
Analyze current compute utilization and headroom.

1. **VM utilization** (last 14 days):
   - Average and P95 CPU across all VMs
   - Average and P95 memory (if available via Log Analytics)
   - Flag VMs consistently > 80% CPU or memory

2. **App Service plan utilization**:
   ```bash
   az appservice plan list --query "[].{name:name, sku:sku.name, workers:numberOfWorkers, maxWorkers:maximumElasticWorkerCount}" -o table
   ```
   - Current instance count vs. max
   - CPU/memory % of the plan
   - Autoscale rules and current headroom

3. **AKS cluster capacity**:
   ```bash
   az aks list --query "[].{name:name, nodeCount:agentPoolProfiles[0].count, maxCount:agentPoolProfiles[0].maxCount, vmSize:agentPoolProfiles[0].vmSize}" -o table
   ```
   - Node count vs. max count
   - Pod density per node
   - Cluster autoscaler configuration

4. **Container Apps**:
   - Current replicas vs. max replicas
   - Autoscale rules (HTTP concurrency, CPU, custom)

### Step 3: Data layer capacity
1. **SQL Database**:
   - DTU/vCore utilization (avg and P95)
   - Storage usage vs. max size
   - Connection count vs. limit
   
2. **Cosmos DB**:
   - RU consumption vs. provisioned (or autoscale max)
   - Storage per container
   - 429 (throttling) rate from metrics

3. **Redis Cache**:
   - Memory usage %
   - Connection count vs. max
   - Server load %

4. **Storage accounts**:
   - Ingress/egress patterns
   - Transaction rates vs. limits
   - Blob count growth rate

### Step 4: Networking capacity
1. **ExpressRoute / VPN Gateway**: Bandwidth utilization %
2. **Application Gateway / Front Door**: Connection count, throughput
3. **NAT Gateway**: SNAT port utilization
4. **DNS zones**: Query volume trends

### Step 5: Growth projection
Based on the data collected:

1. Plot utilization trends for the last 90 days
2. Apply linear projection to estimate when each resource hits 80% and 100%
3. If the user specified a growth factor, multiply current usage and check against limits

Build a risk timeline:

| Resource | Current | At 2x Load | At 5x Load | Hits Limit | Action Needed |
|----------|---------|------------|------------|------------|---------------|
| vCPUs East US | 85/100 | 170 ❌ | 425 ❌ | Now | Quota increase |
| SQL DTU | 60% | 120% ❌ | 300% ❌ | At 1.7x | Scale up tier |
| AKS nodes | 5/10 | 10/10 ⚠️ | 25 ❌ | At 2x | Increase max |
| Redis memory | 45% | 90% ⚠️ | 225% ❌ | At 2.2x | Upgrade SKU |

### Step 6: Quota increase requests
For any quota that needs increasing, generate the correct request command:

```bash
# Request a quota increase via the Azure portal Support + Troubleshooting flow,
# or use the REST API directly:
az rest --method patch \
  --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.Capacity/resourceProviders/Microsoft.Compute/locations/<region>/serviceLimits/StandardDv5Family?api-version=2020-10-25" \
  --body '{"properties":{"limit":{"limitObjectType":"LimitValue","value":<new-limit>}}}'
```

Note: The `az quota` CLI extension is in preview and syntax may change. The REST API approach above is stable. Alternatively, guide users to submit quota requests via the Azure portal: **Help + support > New support request > Service and subscription limits (quotas)**.

### Step 7: Recommendations
For each risk identified:
1. **Immediate**: Quota increase requests, SKU upgrades
2. **Short-term** (1-2 weeks): Autoscale configuration, caching layers
3. **Long-term**: Architecture changes (sharding, async patterns, CDN)

## Expected output

### Capacity summary
```
🏗️ Resources scanned: X
⚠️ Resources at risk (>70% capacity): Y
🔴 Resources critical (>90% capacity): Z
📈 Projected limit breach within 30 days: W resources
```

### Risk matrix
Table with all resources, current utilization, projected utilization at specified growth factor, and recommended action.

### Scaling runbook
Step-by-step actions to prepare for the expected load increase, ordered by priority and dependency.

### Remediation guidance
For each capacity risk identified, include in the output:
1. The specific `az` CLI command to remediate (quota increase, scale-up, autoscale config)
2. Use `GetAzCliHelp` to validate the command syntax before suggesting
3. The official Microsoft Learn documentation link

### References
- Quotas Overview: https://learn.microsoft.com/en-us/azure/quotas/quotas-overview
- VM Sizes: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview
- Autoscale: https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview
- App Service Scaling: https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up
