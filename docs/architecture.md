# Architecture Decisions

Design choices and rationale behind the skills pack.

## Tools & Safety

- **Tools used**: `RunAzCliReadCommands` + `execute_kusto_query` — read-only by design for safety
- **No write operations**: Skills analyze and report; remediation commands are suggested but not auto-executed
- **Remediation guidance**: Each skill includes inline 📖 Microsoft Learn links and instructs the agent to validate CLI commands via `GetAzCliHelp` before suggesting

## Implementation Details

- **Date handling**: Uses Linux shell `$(date -u -d '...' +%Y-%m-%dT%H:%M:%SZ)` syntax (SRE Agent sandbox is Linux)
- **Cost data**: Uses `az costmanagement query` (stable) instead of deprecated `az consumption` commands
- **Quota increases**: Documents REST API approach (`az rest`) since `az quota` CLI is preview/unstable
- **Resource Graph**: Used as fallback when az cli commands require `--resource-group` but subscription-wide scan is needed
- **Kusto queries**: Skills 07 and 08 use `execute_kusto_query` for Log Analytics data (429 throttling, token trends)

## Skill Structure

Each SKILL.md follows a consistent structure:

```
---
name: skill-name
description: >
  Clear description with trigger keywords for intent matching.
tools:
  - RunAzCliReadCommands
  - execute_kusto_query  (if Kusto queries needed)
---

# Skill Title
## Purpose
## When to use this skill
## Pre-check
## Assessment procedure (with checks, commands, scoring tables)
## Scoring
## Expected output
## Remediation guidance
## References
```

## Scoring Philosophy

- Every skill produces a **numeric score** (0–100) with **maturity levels**
- Scores are designed for **trend tracking** — run monthly and compare
- Maturity labels describe **Azure governance state**, not business maturity
- Each check has explicit pass/partial/fail criteria with point values

## Safety Model

Skills operate in a **suggest, don't execute** model:
1. Skills use `RunAzCliReadCommands` — read-only commands only
2. Remediation commands are **printed as suggestions** in the output
3. The agent validates command syntax via `GetAzCliHelp` before suggesting
4. No changes are made to the Azure environment automatically
