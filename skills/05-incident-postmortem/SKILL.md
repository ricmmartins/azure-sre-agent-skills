---
name: incident-postmortem-generator
description: Generate a blameless incident postmortem from the current or recent SRE Agent investigation. Use after an incident is resolved, when asked for a postmortem, RCA report, incident review, or lessons learned document.
tools:
  - execute_kusto_query
  - RunAzCliReadCommands
---

# Incident Postmortem Generator

## Purpose
Automatically generate a structured, blameless postmortem document from the context of a resolved incident investigation. Leverages the timeline, root cause analysis, and mitigations already gathered by the SRE Agent.

## When to use this skill
- User asks for a postmortem after an incident is resolved
- User asks for an RCA report, incident review, or lessons learned
- Automated trigger after incident resolution (via agent hook)
- User asks "write up what just happened"

## Postmortem principles
- **Blameless**: Focus on systems and processes, never individuals
- **Evidence-based**: Every claim backed by data (logs, metrics, timeline)
- **Action-oriented**: Every finding leads to a concrete action item
- **Learning-focused**: What can we improve systemically?

## Generation procedure

### Step 1: Gather incident context
From the current conversation and agent memory, extract:

1. **Incident metadata**:
   - Incident ID (from PagerDuty/ServiceNow if connected)
   - Severity level
   - Duration (detection to resolution)
   - Affected services and resources
   - Impacted users/customers (if known)

2. **Timeline**: Reconstruct from the investigation:
   - When did the issue start? (first anomaly in metrics)
   - When was it detected? (alert fired)
   - When was it acknowledged? (engineer engaged)
   - Key investigation milestones
   - When was mitigation applied?
   - When was full resolution confirmed?

3. **Root cause**: From the agent's RCA:
   - What failed?
   - Why did it fail?
   - Contributing factors

4. **Mitigation applied**: What was done to resolve it?

### Step 2: Assess detection and response
Analyze the incident response quality:

1. **Time to detect (TTD)**: From issue start to alert firing
   - Was this acceptable? Could monitoring have caught it earlier?
   
2. **Time to acknowledge (TTA)**: From alert to human engagement
   - Were on-call rotations effective?

3. **Time to mitigate (TTM)**: From engagement to mitigation
   - Was the runbook adequate? Did the team have the right access?

4. **Time to resolve (TTR)**: Total duration
   - Compare with SLA/SLO targets

### Step 3: Five Whys analysis
Apply the 5 Whys technique to the root cause:

```
1. Why did the service return 500 errors?
   → The application pod was OOM-killed
2. Why was the pod OOM-killed?
   → Memory usage exceeded the 512Mi limit
3. Why did memory usage spike?
   → A new dependency introduced a memory leak in the connection pool
4. Why wasn't the leak caught before production?
   → Load testing doesn't cover the connection pool under sustained load
5. Why doesn't load testing cover this scenario?
   → Load test profiles haven't been updated since the architecture change
```

### Step 4: Generate action items
For each finding, create a SMART action item:

| ID | Action | Owner | Priority | Due date | Status |
|----|--------|-------|----------|----------|--------|
| AI-1 | Add memory-based autoscaling to payment-service | Platform team | High | +7 days | Open |
| AI-2 | Update load test to include sustained connection pool scenarios | QA team | High | +14 days | Open |
| AI-3 | Add memory utilization alert at 80% threshold | SRE team | Medium | +3 days | Open |
| AI-4 | Review connection pool settings across all services | Dev team | Medium | +14 days | Open |

Categories:
- **Prevent recurrence**: Fix the root cause
- **Improve detection**: Better monitoring/alerting
- **Improve response**: Better runbooks/automation
- **Improve resilience**: Architectural improvements

### Step 5: Compile the postmortem document

## Expected output

### Report header (mandatory — use this exact format)

## Incident Postmortem Report

| Field | Value |
|-------|-------|
| Subscription | (name + ID) |
| Incident Date | YYYY-MM-DD |
| Severity | Sev-X |
| Duration | Xh Ym |
| Status | Draft — pending team review |

### Postmortem document structure

```markdown
# Incident Postmortem: [Title]

## Executive summary
[2-3 sentences: what happened, impact, resolution]

## Impact
- **Services affected**: [list]
- **User impact**: [description and scope]
- **SLA/SLO impact**: [metrics affected]
- **Business impact**: [revenue, reputation, etc. if known]

## Timeline (all times UTC)

| Time | Event |
|------|-------|
| HH:MM | First anomaly detected in metrics |
| HH:MM | Alert fired: [alert name] |
| HH:MM | On-call engineer acknowledged |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Mitigation applied |
| HH:MM | Service fully recovered |

## Root cause
[Detailed technical explanation]

## Five Whys
[Analysis chain]

## Detection & response assessment

| Metric | Value | Target | Assessment |
|--------|-------|--------|------------|
| Time to detect | Xm | <5m | ✅/⚠️/❌ |
| Time to acknowledge | Xm | <15m | ✅/⚠️/❌ |
| Time to mitigate | Xm | <30m | ✅/⚠️/❌ |
| Time to resolve | Xm | <2h | ✅/⚠️/❌ |

## What went well
- [Positive aspects of the response]

## What could be improved
- [Areas for improvement]

## Action items
[Table from Step 4]

## Lessons learned
- [Key takeaways for the team]

---
*This postmortem was auto-generated by Azure SRE Agent and should be reviewed by the incident team before publishing.*
```

### Delivery
- Output the full document in the chat
- If Teams connector is available, offer to post to the incident channel
- If ServiceNow/PagerDuty is connected, offer to attach to the incident record
- Suggest scheduling a postmortem review meeting

### References
- Postmortem Culture: https://learn.microsoft.com/en-us/azure/well-architected/operational-excellence/mitigation-strategy
- Azure Resource Health: https://learn.microsoft.com/en-us/azure/service-health/resource-health-overview
- Activity Log: https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log
- Service Health: https://learn.microsoft.com/en-us/azure/service-health/overview
