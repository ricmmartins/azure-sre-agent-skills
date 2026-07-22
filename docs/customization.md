# Customization Guide

Each SKILL.md is self-contained and designed to be easily adapted to your organization's standards.

## Common customizations

### Tagging standards (Compliance & FinOps skills)

Default mandatory tags checked:
```
environment, owner, cost-center, application
```

To change, edit the tag names in:
- `skills/02-compliance-governance/SKILL.md` → Step 3
- `skills/04-finops-intelligence/SKILL.md` → Step 6

Example — adding a `data-classification` tag:
```bash
az resource list --query "[?tags.environment==null || tags.owner==null || tags['data-classification']==null].{name:name, type:type}" -o table
```

### Cost allocation model (FinOps skill)

The default allocation model uses the `cost-center` tag. To change:
1. Edit `skills/04-finops-intelligence/SKILL.md` → Pre-check section
2. Update the default tag name in Step 6
3. Adjust the "Unallocated cost" threshold

### Quota thresholds (Capacity Planning)

Default thresholds:
- 🟢 Green: < 70% utilized
- 🟡 Yellow: 70-90% utilized
- 🔴 Red: > 90% utilized

To adjust, edit `skills/03-capacity-planning/SKILL.md` → Step 1 (change the "flag if > 70%" text).

### Secure Score targets (Defender skill)

Default target: improve by 6+ points per sprint. To change, edit `skills/06-defender-secure-score/SKILL.md` → Expected output section.

### Postmortem template (Incident skill)

The postmortem template is fully customizable. Common changes:
- Add your org's incident severity definitions
- Add links to your incident management tool
- Customize the action item categories
- Add regulatory notification requirements

Edit `skills/05-incident-postmortem/SKILL.md` → Step 5 template.

## Creating variants

You can create specialized variants of any skill. For example:

### Production-only WAF Review
1. Copy `skills/01-well-architected-review/SKILL.md`
2. Change the `name` and `description` to specify production
3. Add a filter: "Only scan resources in resource groups tagged `environment=prod`"
4. Install as a separate skill

### Team-specific FinOps Report
1. Copy `skills/04-finops-intelligence/SKILL.md`
2. Hardcode the team's tag filter
3. Remove the chargeback section (single team doesn't need it)
4. Install as a team-specific skill

## Combining with custom agents

For more complex scenarios, consider creating a **custom agent** that:
1. Runs multiple skills in sequence
2. Maintains state between runs
3. Sends results via email or Teams

Example agent YAML (for WAF Review automation):
```yaml
name: quarterly-waf-reviewer
description: Runs a full Well-Architected review quarterly and posts results to Teams
skills:
  - well-architected-review
connectors:
  - Microsoft Teams
schedule: "0 9 1 */3 *"  # First of every quarter at 9 AM
```

Note: Custom agent creation via YAML is a future capability — check SRE Agent docs for current status.
