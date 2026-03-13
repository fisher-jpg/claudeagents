# Claude Agents — John Fisher

Operational repo for custom Claude skills, agent definitions, workflows, and reference context. Designed to support both autonomous and semi-autonomous AI agents.

## Repo Structure

```
claudeagents/
├── agents/              # Agent definitions (identity, scope, guardrails)
├── skills/              # Single-task instructions (SKILL.md files)
├── workflows/           # Multi-step playbooks chaining skills + tools
├── context/             # Reference docs agents pull on-demand
│   ├── email-templates/
│   └── product-knowledge/
├── memory/              # Persistent behavioral rules for Claude
└── prompts/             # Standalone reusable prompts
```

## How It Fits Together

| Layer | Purpose | When it's loaded |
|---|---|---|
| **agents/** | Defines who the agent is, what it can do, and when to escalate | Once per session |
| **skills/** | How to execute a specific task | On-demand, per task |
| **workflows/** | Orchestrates multi-step sequences across skills and tools | When a workflow triggers |
| **context/** | Reference material (templates, product info, competitive intel) | Selectively, by the active agent or skill |
| **memory/** | Behavioral guardrails that apply across all conversations | Always active via Claude memory |
| **prompts/** | One-shot reusable prompts for ad hoc tasks | On-demand |

## Design Principles

1. **Token efficiency** — Agents load only what they need for the current task. Skills, context, and prompts are self-contained files, not monolithic docs.
2. **Clear autonomy boundaries** — Every agent definition specifies what it can do autonomously, what requires confirmation, and what it should never do.
3. **Composability** — Skills are single-purpose. Workflows compose them. Agents orchestrate workflows. Each layer is independently testable and swappable.
4. **Trust by default, verify at boundaries** — Agents operate freely within their defined scope but escalate at decision points or when crossing into sensitive actions.

## Current Skills

- **`prep-note-summary`** — Converts free-form sales rep prep notes into a structured one-pager demo plan with risk flags, discovery questions, stakeholder-aware demo flow, and info gap checklists

## About

These assets support my work as a Solutions Consultant at Gem, where I use Claude to streamline demo prep, prospect research, and sales enablement workflows.
