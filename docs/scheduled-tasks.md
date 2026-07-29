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

1. Open Azure SRE Agent → **Automation** (left sidebar)
2. Click **+ Create** → **Scheduled task**
3. Configure:
   - **Task name**: e.g., "Weekly Governance Audit"
   - **Response subagent**: (optional) Select a specific subagent if configured
   - **Task details**: The natural language instruction for the agent (include resource identifiers and expected output format)
   - **Frequency**: Daily, Weekly, Monthly, or custom
   - **Time of day**: When to run (timezone shown in UI)
   - **Start on / Repeat until**: Date range for the schedule
   - **Model tier**: General Purpose (default) or higher
   - **Message grouping for updates**: "Use same chat thread" keeps all runs in one thread
   - **Set a run limit**: (optional) Max number of executions
   - **Agent autonomy level**: Autonomous (runs without approval) or Review (asks before acting)
4. Click **Create task**

### Example: Weekly security score check

```
Task name: Weekly Secure Score Monitor
Frequency: Weekly (Friday)
Time of day: 15:00 UTC
Task details: "Check our Defender Secure Score. Compare with last week's score. List the top 3 quick wins we can implement this sprint. If score dropped, explain why."
Agent autonomy level: Autonomous
```

### Example: Monthly FinOps report

```
Task name: Monthly FinOps Report
Frequency: Monthly (1st)
Time of day: 10:00 UTC
Task details: "Generate a full FinOps report for last month. Include cost allocation by the cost-center tag, identify all waste, and compare spend vs. previous month. Highlight any team with >20% increase."
Agent autonomy level: Autonomous
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
1. Check **Automation** → **Scheduled tasks** tab for execution history (Last run, Completed runs columns)
2. Use "Message grouping for updates" set to "Use same chat thread" to keep a running history in one place
3. Review the agent's memory periodically for trend insights

## Cost considerations

Each scheduled task execution consumes agent compute:
- Simple skills (Defender score check): ~500-1000 tokens → minimal cost
- Complex skills (full WAF review): ~5000-10000 tokens → moderate cost
- Use **Model tier** "General Purpose" for routine checks (cheaper) and upgrade only for complex analysis

Tip: Set a **run limit** on experimental tasks to avoid unexpected costs while tuning prompts.
