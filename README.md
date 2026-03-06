# agent-authoring

Skill instruction for GitHub Copilot that teaches it how to write principle-based agents with clear delegation contracts and countable tripwires.

## What this is

A single markdown file (`agent-authoring.md`) passed as a URL to Copilot during agent tuning sessions. It defines how to author both **sub-agents** (delegatees) and **orchestrators** for an autopilot agent system.

## Structure

The document is organized into four audience-tagged sections:

| Section | Audience | Covers |
|---|---|---|
| **I. Shared Foundations** | Both | File structure, status tokens, writing style, file size rules |
| **II. Writing Sub-Agents** | Sub-agent authors | Objective anchors, breadcrumbs, roles, contracts, brief validation, execution rules with tripwires |
| **III. Writing Orchestrators** | Orchestrator authors | Brief completeness, monitoring, recovery, consultation, adaptive freedom |
| **IV. Common Mistakes** | Both (tagged per audience) | Failure patterns and fixes, tagged `[sub-agent]`, `[orchestrator]`, or `[both]` |

## Key principles

- **Principles guide, tripwires enforce.** Every execution rule has a principle (why) and a tripwire (a countable limit). Principles require interpretation; tripwires fire automatically. An agent that hits a tripwire emits a status token and stops — no deliberation.
- **Sub-agents are tightly constrained.** Objective anchor before work, breadcrumbs after every response, brief validation before first read, no mid-work exploration, return `LOST` when stuck.
- **Orchestrators have adaptive freedom.** They follow guidelines, not rigid gates. They're accountable for outcomes — if sub-agents produce verified artifacts matching the objective, the path was acceptable.
- **Brief completeness prevents drift.** Incomplete briefs are the #1 cause of sub-agents getting lost. Every brief needs: objective, input files, constraints (including build/test command), done condition, expected tokens. Sub-agents validate briefs and return `GAP` with the missing field named.
- **Status tokens replace prose parsing.** Machine-readable first-word tokens (`GO`, `COMPLETE`, `CONCERN`, `LOST`, etc.) let the orchestrator gate without interpreting natural language.
- **Verifiable over aspirational.** "What Good Looks Like" sentences must pass a litmus test: could a machine or grep command confirm it? "All tests pass" — verifiable. "Code is clean" — aspirational.

## Usage

Pass the raw URL of `agent-authoring.md` to Copilot during an agent tuning session. No companion files needed — the document is self-contained.

## Example agents

The `.github/agents/` directory contains a reference set of agents authored against this spec:

| Agent | Type | Role |
|---|---|---|
| `architect` | Orchestrator | Decomposes features into micro-briefs, delegates to sub-agents, monitors for drift |
| `developer` | Sub-agent (write) | Implements code changes with build verification |
| `tester` | Sub-agent (write) | Writes and runs unit tests |
| `documenter` | Sub-agent (write) | Writes markdown documentation with symbol-completeness verification |
| `reviewer` | Sub-agent (read-only) | Reviews code changes, returns numbered findings or approval |