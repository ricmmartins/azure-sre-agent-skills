# Scheduled Tasks Setup

Azure SRE Agent supports **scheduled tasks** that run skills automatically on a cadence. This turns reactive skills into proactive monitoring.

## Recommended schedules

| Skill | Cadence | Best day/time | Prompt to schedule |
|-------|---------|---------------|-------------------|
| Compliance & Governance | Weekly | Monday 9 AM | "Run a governance audit and report findings" |
| FinOps Intelligence | Monthly | 1st business day, 10 AM | "Generate the monthly FinOps report with cost allocation" |
| Capacity Planning | Bi-weekly | Every other Wednesday | "Check capacity headroom for all production workloads" |
| Defender Secure Score | Weekly | Friday 3 PM | "Check our secure score and identify top improvements" |
| Well-Architected Review | Quarterly | 1st Monday of quarter | "Run a full Well-Architected review" |

## Setting up scheduled tasks

### Via the SRE Agent Portal

1. Open Azure SRE Agent → **Scheduled Tasks**
2. Click **+ New scheduled task**
3. Configure:
   - **Name**: e.g., "Weekly Governance Audit"
   - **Schedule**: Select cadence (daily/weekly/monthly/custom cron)
   - **Prompt**: The natural language instruction for the agent
   - **Notification**: Where to send results (email, Teams, or store in memory)
4. Click **Create**

### Example: Weekly security score check

```
Name: Weekly Secure Score Monitor
Schedule: Every Friday at 15:00 UTC
Prompt: "Check our Defender Secure Score. Compare with last week's score. List the top 3 quick wins we can implement this sprint. If score dropped, explain why."
Notify: Teams channel #security-ops
```

### Example: Monthly FinOps report

```
Name: Monthly FinOps Report
Schedule: 1st of every month at 10:00 UTC
Prompt: "Generate a full FinOps report for last month. Include cost allocation by the cost-center tag, identify all waste, and compare spend vs. previous month. Highlight any team with >20% increase."
Notify: Email to finops@company.com
```

## Tips for effective scheduled tasks

### Be specific in prompts
❌ "Check costs"
✅ "Identify cost savings opportunities across all production subscriptions. Focus on orphaned resources and rightsizing. Only report findings with >$50/month potential savings."

### Include comparison context
❌ "Run security check"
✅ "Check our Defender Secure Score and compare with the last run. Report new findings, resolved findings, and score trend."

### Set appropriate cadences
- **Don't over-schedule**: Running cost analysis daily wastes agent compute (costs change slowly)
- **Do schedule before events**: Add an ad-hoc capacity check before known traffic spikes
- **Match business rhythm**: FinOps monthly aligns with billing cycles; governance weekly matches sprint cadence

### Combine with memory
The SRE Agent has memory. Scheduled tasks build up a history:
- "Compare this week's compliance score with last week's"
- "Are the same issues repeating from the last 3 capacity checks?"
- "Has our secure score trend been improving over the last month?"

## Monitoring scheduled task results

After each run:
1. Check **Scheduled Tasks** → **History** for execution logs
2. Set up Teams/email notifications for critical findings
3. Review the agent's memory periodically for trend insights

## Cost considerations

Each scheduled task execution consumes AAU (Agent Activity Units):
- Simple skills (Defender score check): ~500-1000 tokens → minimal cost
- Complex skills (full WAF review): ~5000-10000 tokens → moderate cost

Estimate: A weekly cadence across all 6 skills ≈ 4-6 AAU/month at variable rates. Check your AAU allocation in **Settings** → **Usage & billing**.
