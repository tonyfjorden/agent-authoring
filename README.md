# Agent authoring

Skill instruction for GitHub Copilot Cli. Teaches it to write agents with delegation contracts, status tokens, and countable tripwires.

This skill instruction is still under heavy development and feedback is appriciated!

## What this is

One markdown file (`agent-authoring.md`). Pass the raw URL to Copilot during agent tuning. Self-contained — no companion files.

## Structure

| Section | Covers |
|---|---|
| **I. Shared Foundations** | File structure, status tokens, writing style, file size, infra resilience |
| **II. Agent Archetypes** | Eight archetypes (scout, implementer, tester, reviewer, advisor, documentarian, auditor, orchestrator), tool rules, split guidance, profiles |
| **III. Writing Sub-Agents** | Role, delegation contract, runtime behaviors, tripwires, brief validation |
| **IV. Writing Orchestrators** | Required sections, brief completeness, monitoring, fallback routing, recovery |
| **V. Common Mistakes** | Tagged `[sub-agent]`, `[orchestrator]`, or `[both]` |

## How it works

- **Tripwires enforce rules.** Each execution rule has a countable limit. Hit it → emit token → stop.
- **Status tokens replace prose.** First word of every response is a machine-readable token (`GO`, `CONCERN`, `LOST`, etc.).
- **Sub-agents are constrained.** One pass, one verify. No mid-work exploration. `LOST` when stuck.
- **Orchestrators have freedom.** Accountable for outcomes, not process.
- **Briefs prevent drift.** Five fields: objective, input files, constraints, done condition, expected token.