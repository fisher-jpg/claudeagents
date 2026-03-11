# Context

Reference documents that agents and skills pull in on-demand.

## Subdirectories

- **`email-templates/`** — Reusable communication templates
- **`product-knowledge/`** — Gem product info, competitive intel, feature details

## Token efficiency
Agents should load only the context files relevant to their current task. Never load the entire directory. Each file should be self-contained and focused on one topic.

## Naming convention
Use descriptive names: `account-intro-template.md`, `gem-vs-ashby.md`, `greenhouse-integration-faq.md`
