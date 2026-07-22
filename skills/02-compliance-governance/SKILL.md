---
name: compliance-governance-audit
description: Audit Azure environment for compliance and governance posture. Checks Azure Policy, RBAC, tagging, resource locks, naming conventions, and regulatory alignment. Use when asked about compliance, governance, audit readiness, or policy violations.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query
---

# Compliance & Governance Audit

## Purpose
Assess the governance posture of Azure subscriptions by auditing policies, RBAC assignments, tagging standards, resource locks, and regulatory alignment. Produce a compliance scorecard with remediation steps.

## When to use this skill
- User asks "are we compliant?"
- User asks about governance, policy violations, or audit readiness
- Pre-audit preparation (SOC2, ISO 27001, HIPAA, etc.)
- Monthly governance review

## Pre-check
Confirm with the user:
- Scope: which subscriptions/management groups
- Compliance framework (if any): SOC2, ISO 27001, HIPAA, PCI-DSS, CIS Benchmarks, or general best practices
- Any known exceptions or waivers

## Audit procedure

### Step 1: Azure Policy compliance
Check overall policy compliance state.

```bash
az policy state summarize --query "value[].{policy:policyDefinitionName, nonCompliant:nonCompliantResources}" -o table
```

1. List all non-compliant resources grouped by policy
2. Identify policies in "audit" mode that should be "deny"
3. Check for orphaned policy assignments (assigned but no effect)
4. Verify initiative assignments for regulatory frameworks (CIS, NIST, etc.)

Report:
- Total policies assigned
- % compliant
- Top 5 violated policies with resource count

### Step 2: RBAC review
Audit role assignments for least-privilege violations.

```bash
az role assignment list --all --query "[].{principal:principalName, role:roleDefinitionName, scope:scope}" -o table
```

1. **Owner count**: Flag if > 3 Owners at subscription level
2. **Contributor sprawl**: List all Contributor assignments — flag service principals with Contributor when a custom role would suffice
3. **Classic admins**: Check for legacy co-administrators
   ```bash
   az role assignment list --include-classic-administrators -o table
   ```
4. **Guest users with privileged roles**: Flag external identities with Owner/Contributor
5. **Stale assignments**: Cross-reference with sign-in logs — flag principals that haven't signed in for 90+ days
6. **Custom roles**: Review custom role definitions for overly broad permissions (e.g., `*` actions)

### Step 3: Tagging compliance
Check mandatory tags across all resources.

Define expected mandatory tags (confirm with user or use defaults):
- `environment` (prod/staging/dev/test)
- `owner` (team or individual)
- `cost-center` (billing code)
- `application` (workload name)

```bash
az resource list --query "[?tags.environment==null || tags.owner==null].{name:name, type:type, rg:resourceGroup, tags:tags}" -o table
```

Report:
- % of resources with all mandatory tags
- Top offending resource groups
- Resources with no tags at all

### Step 4: Resource locks
Check critical resources for delete/read-only locks.

```bash
az lock list --query "[].{name:name, level:level, resource:resourceId}" -o table
```

1. Verify production databases have delete locks
2. Verify production storage accounts have delete locks
3. Verify networking resources (VNets, ExpressRoute) have locks
4. Flag production resources without any lock

### Step 5: Naming conventions
Analyze resource naming patterns.

1. Extract all resource names and types
2. Check against Azure naming conventions (e.g., `rg-`, `vnet-`, `vm-`, `st`, `kv-`)
3. Flag resources that don't follow a consistent pattern
4. Report % compliance with naming standards

### Step 6: Network governance
1. Check for resources with public endpoints that should be private
2. Verify NSG flow logs are enabled
3. Check for Network Watcher in all active regions
4. Verify DDoS protection on VNets with public-facing resources

### Step 7: Diagnostic settings
1. Check that all critical resources have diagnostic settings enabled
2. Verify logs flow to a central Log Analytics workspace
3. Check Activity Log export at subscription level

```bash
az monitor diagnostic-settings subscription list --subscription <id> -o table
```

## Scoring model

| Rating | Meaning |
|--------|---------|
| 🟢 **Compliant** | Meets standard, no action needed |
| 🟡 **Partial** | Partially implemented, needs improvement |
| 🔴 **Non-compliant** | Missing or misconfigured, action required |

## Expected output

### Governance scorecard

| Area | Status | Compliant | Partial | Non-compliant | Score |
|------|--------|-----------|---------|---------------|-------|
| Azure Policy | 🟡 | X | Y | Z | % |
| RBAC | 🔴 | ... | ... | ... | % |
| Tagging | 🟡 | ... | ... | ... | % |
| Resource Locks | 🔴 | ... | ... | ... | % |
| Naming | 🟢 | ... | ... | ... | % |
| Network Gov. | 🟡 | ... | ... | ... | % |
| Diagnostics | 🟡 | ... | ... | ... | % |
| **Overall** | | | | | **%** |

### Critical findings (act now)
Items that represent immediate risk or audit failure.

### Remediation plan
For each finding:
- What's wrong
- Why it matters
- How to fix it (with az cli command or portal steps)
- Use `GetAzCliHelp` to validate the command syntax before suggesting
- Include the official Microsoft Learn documentation link
- Estimated effort
- Priority (Critical/High/Medium/Low)

### References
- Azure Policy: https://learn.microsoft.com/en-us/azure/governance/policy/overview
- RBAC Best Practices: https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices
- Resource Tagging: https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources
- Resource Locks: https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources
- Naming Conventions: https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/naming-and-tagging

### Regulatory mapping (if framework specified)
Map findings to specific controls (e.g., SOC2 CC6.1, ISO 27001 A.9.2.3).
