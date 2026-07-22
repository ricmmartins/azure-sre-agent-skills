---
name: finops-intelligence
description: Comprehensive FinOps analysis combining cost optimization, waste identification, and chargeback reporting. Use when asked about reducing Azure spend, finding unused resources, cost per team, chargeback, showback, cost anomalies, rightsizing, or monthly cost review.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# FinOps Intelligence

## Purpose
Unified cost intelligence skill that identifies savings opportunities, tracks cost trends, and generates chargeback/showback reports by team or project. Answers both "where can we save?" and "who spent what?"

## When to use this skill
- User asks "why did our bill go up?"
- User asks for cost optimization or savings opportunities
- User asks "how much did team X spend this month?"
- User asks for chargeback, showback, or cost allocation report
- Monthly proactive cost review or FinOps cadence

## Pre-check
Confirm with the user:
- **Scope**: Which subscriptions to scan (all or specific ones)
- **Time range**: For trend analysis (default: last 3 months)
- **Allocation model** (for chargeback): Which tag to use for cost splitting?
  - `cost-center` tag (most common)
  - `owner` or `team` tag
  - `application` or `project` tag
  - Resource group naming convention (e.g., `rg-teamname-*`)
- **Exclusions**: Dev/test subscriptions, sandbox resource groups

## Analysis procedure

### Step 1: Cost trend overview
Get the big picture using Azure Cost Management.

```bash
# Current month cost by service (last 30 days)
az costmanagement query --type ActualCost --timeframe MonthToDate \
  --scope "subscriptions/<sub-id>" \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="ServiceName" type="Dimension" \
  -o table
```

```bash
# Previous month for comparison
az costmanagement query --type ActualCost --timeframe TheLastMonth \
  --scope "subscriptions/<sub-id>" \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="ServiceName" type="Dimension" \
  -o table
```

If `az costmanagement` is unavailable, use the REST API:
```bash
az rest --method post \
  --url "https://management.azure.com/subscriptions/<sub-id>/providers/Microsoft.CostManagement/query?api-version=2023-11-01" \
  --body '{"type":"ActualCost","timeframe":"MonthToDate","dataset":{"aggregation":{"totalCost":{"name":"Cost","function":"Sum"}},"grouping":[{"type":"Dimension","name":"ServiceName"}]}}'
```

Summarize:
- Total spend this month vs. last month (% change)
- Top 5 services by spend
- Top 5 resource groups by spend
- Any spending anomalies (day-over-day spikes > 20%)

### Step 2: Orphaned resources (waste)
Find resources consuming cost with no active use.

| Check | Command | Savings signal |
|-------|---------|----------------|
| Unattached managed disks | `az disk list --query "[?managedBy==null]"` | Disk cost per month |
| Unused public IPs | `az network public-ip list --query "[?ipConfiguration==null]"` | ~$3.65/month each |
| Stopped but allocated VMs | `az vm list -d --query "[?powerState=='VM deallocated']"` | Disk + IP cost still billed |
| Unused App Service plans | `az appservice plan list --query "[?numberOfSites==0]" --resource-group <rg>` | Full plan cost |
| Old snapshots (>90 days) | `az snapshot list --query "[?timeCreated<'$(date -u -d '90 days ago' +%Y-%m-%dT%H:%M:%SZ)']"` | Storage cost |
| Unused NAT Gateways | `az network nat gateway list` cross-ref with subnets | ~$32/month each |
| Empty resource groups | `az group list` then check member count | Organizational waste |

Note on date handling: Use shell variable substitution for date comparisons:
- Linux/macOS: `$(date -u -d '90 days ago' +%Y-%m-%dT%H:%M:%SZ)`
- The SRE Agent sandbox runs Linux, so the above syntax is valid.

### Step 3: Rightsizing opportunities
Identify over-provisioned resources.

1. **VMs with low CPU** (< 5% avg over 14 days):
   Query Azure Monitor metrics for `Percentage CPU` across all VMs
2. **VMs with low memory** (< 10% avg):
   Query Log Analytics for memory counters if available
3. **Over-provisioned App Service plans**:
   Check CPU and memory % across the plan — if consistently < 20%, suggest downgrade
4. **Over-provisioned databases**:
   Check DTU/vCore utilization — if < 20%, suggest lower tier

For each, calculate:
- Current SKU and monthly cost
- Recommended SKU and monthly cost
- **Estimated monthly savings**

### Step 4: Reservation & savings plan opportunities
1. List VMs running 24/7 for > 30 days — candidates for Reserved Instances (up to 72% savings)
2. List databases running 24/7 — candidates for reserved capacity
3. Check if Azure Savings Plans could apply to compute spend

### Step 5: Storage optimization
1. Check storage accounts for access tier usage:
   ```bash
   az storage account list --query "[].{name:name, accessTier:accessTier, kind:kind}" -o table
   ```
2. Identify blobs that haven't been accessed in 90+ days — candidates for Cool/Archive tier
3. Check for lifecycle management policies — suggest if missing
4. Check for redundancy over-provisioning (GRS when LRS would suffice for non-critical data)

### Step 6: Cost allocation (chargeback/showback)
Group costs by the user's chosen allocation model.

**By tag** (preferred):
- Group all resources by the chosen tag value
- Aggregate costs per tag value
- Track untagged resources separately as "Unallocated"

**By resource group** (fallback):
- Parse resource group names for team/project identifiers
- Group and aggregate accordingly

**Shared costs** (identify and handle):
- Resources used by multiple teams (e.g., shared AKS cluster, shared networking)
- Flag these separately — suggest allocation keys (even split, usage-based, or headcount-based)

For each team/project:
- This period vs. previous period: $ change and % change
- Top cost driver (which service drove the change?)
- Flag anomalies (> 30% increase without known cause)

### Step 7: Efficiency metrics
Calculate per-team efficiency indicators:
- **Cost per resource**: Total spend / number of resources
- **Compute waste ratio**: Cost of idle/underutilized resources / total compute cost
- **Tag compliance**: % of team's resources properly tagged

## Expected output

### Executive summary
```
💰 Total Azure spend (this month): $XX,XXX
📈 vs. last month: +/-X% ($+/-X,XXX)
🗑️ Waste identified: $X,XXX/month recoverable
👥 Teams/projects tracked: X
🏷️ Allocation rate: X% (Y% unallocated)
🔍 Issues found: X critical, Y high, Z medium
```

### Savings breakdown table

| Category | Finding | Current Cost/mo | Savings/mo | Priority | Action |
|----------|---------|----------------|------------|----------|--------|
| Orphaned | 3 unattached disks | $45 | $45 | High | Delete or snapshot+delete |
| Rightsizing | 2 VMs at < 5% CPU | $380 | $190 | High | Resize D4s_v5 → B2ms |
| Reservations | 5 VMs running 24/7 | $1,200 | $864 | Medium | 3yr RI |
| Storage | No lifecycle policies | $200 | $80 | Medium | Add cool tier policy |
| **Total** | | | **$X,XXX** | | |

### Cost allocation table

| Team / Project | This Period | Last Period | Change | % Change | Top Service | % of Total |
|----------------|------------|-------------|--------|----------|-------------|------------|
| Platform | $5,200 | $4,800 | +$400 | +8.3% | Compute | 32% |
| Product API | $3,800 | $3,100 | +$700 | +22.6% ⚠️ | Databases | 24% |
| Unallocated | $1,300 | $800 | +$500 | +62.5% 🔴 | Mixed | 8% |
| **Total** | **$16,000** | **$14,300** | **+$1,700** | **+11.9%** | | **100%** |

### Quick wins (implement today)
Top 3 actions that save the most with the least effort.

### Requires planning
Actions that need architecture review or stakeholder approval.

### Unallocated cost remediation
List untagged resources with suggested owner and tagging command:
```bash
az resource tag --ids <resource-id> --tags team=<team> cost-center=<cc>
```

### Remediation guidance
For each cost finding, include in the output:
1. The specific `az` CLI command to remediate (suggest only — do not execute)
2. Use `GetAzCliHelp` to validate the command syntax before suggesting
3. The official Microsoft Learn documentation link

### References
- Cost Management: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management
- Azure Advisor Cost: https://learn.microsoft.com/en-us/azure/advisor/advisor-cost-recommendations
- Reserved Instances: https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/save-compute-costs-reservations
- Orphaned Resources: https://learn.microsoft.com/en-us/azure/advisor/advisor-reference-cost-recommendations
