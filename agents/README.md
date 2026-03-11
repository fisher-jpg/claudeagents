# Agents

Agent definition files live here. Each `.md` file defines a single agent:

- **Identity**: What the agent is and what it does
- **Scope**: What tools, connectors, and skills it can access
- **Guardrails**: What it should never do without approval
- **Escalation rules**: When to pause and ask the user
- **Autonomy level**: Fully autonomous vs. semi-autonomous behaviors

## Naming convention
`[agent-name].md` — e.g., `demo-prep-agent.md`

## Template
See below for the recommended structure of an agent definition file:

```markdown
# Agent: [Name]

## Purpose
One-sentence description of what this agent does.

## Autonomy Level
- **Autonomous**: [list of actions the agent can take without asking]
- **Semi-autonomous**: [list of actions that require user confirmation]
- **Never**: [list of actions the agent must never take]

## Tools & Connectors
Which integrations this agent can use (e.g., Slack, Gmail, Google Drive, Notion).

## Skills
Which skills from `/skills/` this agent can invoke.

## Context
Which files from `/context/` this agent should load, and when.

## Escalation Rules
When to stop and check in with the user.

## Example Triggers
Sample prompts or conditions that should activate this agent.
```
