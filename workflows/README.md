# Workflows

Multi-step playbooks that chain skills, tools, and context together.

Each workflow defines:
- **Trigger**: What kicks it off
- **Steps**: Ordered sequence of actions (skill calls, tool usage, context loading)
- **Decision points**: Where the agent pauses for user input vs. proceeds autonomously
- **Outputs**: What gets produced at the end

## Naming convention
`[workflow-name].md` — e.g., `new-demo-request.md`

## Template

```markdown
# Workflow: [Name]

## Trigger
What activates this workflow.

## Steps
1. [Action] — skill/tool used, inputs, outputs
2. [Action] — ...
3. **Decision point**: [What to ask the user]
4. [Action] — ...

## Outputs
What this workflow produces.

## Autonomy
Which steps are autonomous vs. require confirmation.
```
