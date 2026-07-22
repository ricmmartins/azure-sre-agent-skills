# Installation Guide

## Prerequisites

### Azure SRE Agent access
- Azure SRE Agent must be enabled on your subscription
- [Enable Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/)

### Required Azure roles
| Skill | Minimum role required |
|-------|---------------------|
| Well-Architected Review | Reader |
| Compliance & Governance | Reader + Security Reader |
| Capacity Planning | Reader |
| FinOps Intelligence | Reader + Cost Management Reader |
| Incident Postmortem | Reader (+ Log Analytics Reader for KQL queries) |
| Defender Secure Score | Security Reader |

### Azure CLI extensions (auto-available in SRE Agent sandbox)
- `az costmanagement` — for cost queries
- `az security` — for Defender/Secure Score
- `az graph` — for Azure Resource Graph queries

## Step-by-step installation

### Option 1: Install via SRE Agent Portal (recommended)

1. Open the [Azure SRE Agent portal](https://portal.azure.com/#view/Microsoft_Azure_SREAgent/SREAgentBlade)
2. Navigate to **Builder** → **Skills**
3. Click **+ Create skill**
4. For each skill you want to install:
   - Copy the **entire contents** of the `SKILL.md` file (including the `---` YAML front matter)
   - Paste into the skill editor
   - Click **Save**

### Option 2: Install via API

```bash
# Clone this repo
git clone https://github.com/ricmmartins/azure-sre-agent-skills.git
cd azure-sre-agent-skills

# For each skill, use the SRE Agent API to create it
# (API documentation pending from Microsoft)
```

## Verifying installation

After installing, test each skill by asking the SRE Agent:

| Skill | Test prompt |
|-------|------------|
| WAF Review | "Run a Well-Architected review on my subscription" |
| Compliance | "Check our governance and compliance posture" |
| Capacity | "Check our quota and capacity utilization" |
| FinOps | "Find cost optimization opportunities" |
| Postmortem | "Generate a postmortem template" |
| Defender | "What's our Defender Secure Score?" |

The agent should automatically load the relevant skill based on your question.

## Troubleshooting

### Skill not activating
- Check that the skill's `description` field contains keywords matching your question
- Verify the skill is saved (check Builder → Skills list)
- Remember: max 5 skills loaded concurrently (oldest auto-unloads)

### Permission errors
- Ensure your SRE Agent has the required roles (see Prerequisites above)
- For cost queries, `Cost Management Reader` is often missing — assign it at subscription scope

### Command failures
- Some commands require `--resource-group` — the skill procedures note this
- If `az costmanagement` isn't available, the FinOps skill falls back to `az rest` with the Cost Management REST API
- Date calculations use Linux syntax (`date -u -d '...'`) which works in the SRE Agent sandbox

## Uninstalling

1. Open **Builder** → **Skills**
2. Find the skill to remove
3. Click **Delete**

The skill stops activating immediately.
