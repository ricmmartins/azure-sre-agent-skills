# Azure SRE Agent — Proactive Operations Skills Pack

> Custom skills for [Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/) that fill gaps in its built-in capabilities, focused on **proactive governance**, **cost intelligence**, and **architecture quality**.

> ⚠️ **Community project**: These skills are not officially maintained by Microsoft. They have been tested on a live Azure SRE Agent instance but may require adjustments as the platform evolves. Contributions and bug reports are welcome!

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 🎯 What is this?

Azure SRE Agent is Microsoft's AI-powered site reliability agent with 40+ MCP connectors, sandboxed execution, and memory. While it excels at **reactive** incident response (investigating alerts, running diagnostics), it lacks **proactive** operational skills out of the box.

This pack adds **8 custom skills** that transform your SRE Agent into a proactive operations partner:

| # | Skill | What it does | Value |
|---|-------|-------------|-------|
| 01 | [Well-Architected Review](skills/01-well-architected-review/SKILL.md) | 5-pillar WAF assessment with scoring | Architecture quality gate |
| 02 | [Compliance & Governance](skills/02-compliance-governance/SKILL.md) | Policy, RBAC, tagging, locks, naming audit | Audit readiness |
| 03 | [Capacity Planning](skills/03-capacity-planning/SKILL.md) | Quota utilization, growth projection, scaling prep | Prevent capacity outages |
| 04 | [FinOps Intelligence](skills/04-finops-intelligence/SKILL.md) | Cost optimization + chargeback combined | Reduce waste, track spend |
| 05 | [Incident Postmortem](skills/05-incident-postmortem/SKILL.md) | Blameless postmortem generator (5 Whys) | Accelerate learning loops |
| 06 | [Defender Secure Score](skills/06-defender-secure-score/SKILL.md) | Secure Score monitoring + improvement plan | Continuous security improvement |
| 07 | [Digital Native Governance](skills/07-digital-native-governance/SKILL.md) | Startup governance maturity check (15 checks, scored 0–100) | Enterprise readiness for startups |
| 08 | [AI Foundry & OpenAI Posture](skills/08-ai-foundry-posture/SKILL.md) | Security, reliability & cost posture for Azure OpenAI/Foundry (15 checks) | AI workload production readiness |

## 📸 Example output

Here's what Skill 08 (AI Foundry Posture) looks like when it runs:

> **AI Foundry & OpenAI Posture Check** — Score: **52 / 100** — Level: 🟡 **Developing**

| Category | Score | Max | Status |
|----------|-------|-----|--------|
| 🔐 Security | 18 | 35 | 🟡 |
| ⚙️ Reliability | 17 | 30 | 🟡 |
| 💰 Cost & Efficiency | 12 | 25 | 🟡 |
| 🏗️ Architecture | 5 | 10 | 🟡 |
| **TOTAL** | **52** | **100** | **🟡** |

Each check includes a finding, severity, and remediation command with a 📖 link to Microsoft Learn docs.

## 👤 Who is this for?

| Persona | Use case |
|---------|----------|
| **Solution Engineer** | Run a governance/WAF review before a customer workshop |
| **SRE / Platform Engineer** | Weekly automated checks on security, cost, and compliance |
| **CTO / Engineering Lead** | Pre-audit maturity assessment ("are we production-ready?") |
| **FinOps Practitioner** | Monthly cost optimization + chargeback reports |

## 🚀 Quick start

### Prerequisites
- Azure SRE Agent enabled on your subscription ([docs](https://learn.microsoft.com/en-us/azure/sre-agent/) | [portal](https://sre.azure.com/docs))
- At least **Reader** role on target subscriptions
- For cost skills: **Cost Management Reader** role

### Installation

1. Open the **Azure SRE Agent portal** → **Builder** → **Skills**
2. Click **+ Create skill**
3. Copy the contents of any `SKILL.md` file from this repo
4. Paste into the skill editor
5. Save — the skill auto-activates based on conversational intent

Repeat for each skill you want to install.

### Usage

Once installed, just ask naturally:

```
"Run a Well-Architected review on my production subscription"
"Are we compliant with our governance standards?"
"Can our infrastructure handle 3x traffic for Black Friday?"
"Where can we save money? And how much did each team spend?"
"Write up a postmortem for what just happened"
"What's our secure score and how do we improve it?"
"Are we production-ready? Run a governance maturity check"
"Is our OpenAI deployment secure? Check our AI Foundry posture"
```

The SRE Agent matches your intent to the skill's description and activates it automatically.

## 📋 Skill details

### Priority tiers

- **P0 (must-have)**: Capacity Planning, Compliance & Governance — genuine gaps with no built-in equivalent
- **P1 (high-value)**: WAF Review, FinOps Intelligence, Digital Native Governance, AI Foundry Posture — significant value beyond built-ins
- **P2 (nice-to-have)**: Incident Postmortem, Defender Secure Score — lighter differentiation but useful

### Concurrency note

Azure SRE Agent supports max **5 concurrent skills** (oldest auto-unloads). This pack has 8 skills, but they're designed for different use cases — you'll rarely trigger more than 2-3 simultaneously.

## 🔧 Customization

Each SKILL.md is self-contained. Common customizations:

- **Adjust tagging standards** in Compliance skill (Step 3) to match your organization
- **Change cost allocation tags** in FinOps Intelligence (Pre-check section)
- **Set quota thresholds** in Capacity Planning (default: flag at 70%)
- **Customize postmortem template** to match your org's incident process
- **Adjust governance checks** in Digital Native Governance for your organization's maturity stage
- **Configure PTU thresholds** in AI Foundry Posture (default: 70–85% optimal utilization)

See [docs/customization.md](docs/customization.md) for detailed guidance.

## ⏰ Automation with Scheduled Tasks

These skills become even more powerful when combined with SRE Agent's **Scheduled Tasks** feature:

| Skill | Recommended cadence | Trigger phrase |
|-------|-------------------|----------------|
| Compliance & Governance | Weekly | "Run governance audit" |
| FinOps Intelligence | Monthly (1st business day) | "Generate monthly cost report" |
| Capacity Planning | Bi-weekly | "Check capacity headroom" |
| Defender Secure Score | Weekly | "Check secure score and suggest improvements" |
| Digital Native Governance | Monthly | "Run startup governance maturity check" |
| AI Foundry Posture | Bi-weekly | "Check our OpenAI posture and token consumption" |

See [docs/scheduled-tasks.md](docs/scheduled-tasks.md) for setup instructions.

## 📐 Architecture decisions

See [docs/architecture.md](docs/architecture.md) for detailed design choices, safety model, and skill structure conventions.

## 🤝 Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Ideas for new skills:
- SLA/SLO monitoring dashboard
- Network topology validator
- Migration readiness assessment
- Disaster recovery drill runner
- Container Apps / AKS health check
- Data pipeline monitoring (Data Factory, Synapse)

## 📄 License

[MIT](LICENSE) — use freely, attribute if you like.

---

Built by [@ricmmartins](https://github.com/ricmmartins) | Powered by [Azure SRE Agent](https://sre.azure.com/docs)
