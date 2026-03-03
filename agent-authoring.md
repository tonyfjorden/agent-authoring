# Agent Authoring

Instructions for writing new agents that work correctly in GitHub Copilot's agent system — particularly as delegatees under an autopilot orchestrator.

---

## File structure

Agents live in `.github/agents/` and use the `.agent.md` extension. Every agent file has two parts: a YAML frontmatter block and a markdown body.

```markdown
---
name: my-agent
description: One sentence. What this agent does and when to invoke it.
tools:
  - read_file
  - create_file
  - replace_string_in_file
  - run_in_terminal
  - file_search
  - grep_search
---

# My Agent

## Role
...
```

### Frontmatter rules

- `name` — lowercase, hyphenated, matches the filename stem. Used as the `#my-agent` invocation handle.
- `description` — this is what the orchestrator reads when deciding whether to invoke you. Be specific about the domain. Vague descriptions cause incorrect routing.
- `tools` — declare only what the agent actually needs.

---

## Body structure

Every agent body follows this section order:

```
## Role
## How You Work
## Delegation Contract    ← required for every agent except the orchestrator
## Raising Concerns
## What Good Looks Like
## Skills
```

Do not invent new section names. Consistent structure lets the orchestrator scan for contracts and gates without parsing free prose.

---

## Role section

One paragraph. Answers three questions:
1. What domain does this agent own?
2. What does it produce?
3. What does it explicitly **not** do?

The third question is mandatory. A role without a scope ceiling will drift — the agent will accept briefs it should reject, produce output outside its domain, and corrupt the pipeline. If the answer to "what does it not do?" is obvious, state it anyway. Explicit beats implicit.

**Parallelization declaration (required):** State whether the agent is safe to parallelize.
- **Analysis-only agents** (read-only, no side effects): *"You are analysis-only and safe to parallelize. The architect may invoke you alongside other agents simultaneously."*
- **Write agents** (mutate state, create/modify files): *"You are a write agent. The architect runs you sequentially — never in parallel with another write agent."*

This declaration is required in every agent. Omitting it forces the orchestrator to guess.

---

## Delegation Contract

The contract defines the exact handshake so the orchestrator can gate transitions without parsing prose.

```markdown
## Delegation Contract

| Phase | You receive | You return |
|---|---|---|
| Phase name | What the orchestrator passes in | What you return + status token |

Return status: `TOKEN_A`, `TOKEN_B`, or `CONCERN`.
```

### Contract gates

Every delegation contract must enforce hard limits to prevent overthinking loops:
- **Write agent briefs** should trigger max 2–3 API requests before first tangible action (build, test, file write, code commit).
- **Simple reviews** (≤5 checks) max 1 API request; complex reviews max 2 API requests.
- **Requirements review** max 2 API requests before returning GO/NO-GO.
- **Test writing** max 3 API requests before returning test files.
- If an agent exceeds these limits mid-execution, the orchestrator must terminate as `AGENT OVERTHINKING`, investigate (brief incomplete vs. agent design failure), and re-invoke with corrected input.

**Progress-based gate (required):**
- Do not judge overthinking by request count alone. Judge by **request-to-evidence ratio**.
- For write agents, require evidence early: first build/test/verify command must appear within the first 2–3 requests.
- After first evidence, require recurring evidence: at least one verify/build/test action every 3–5 requests.
- If requests continue without new evidence (no file write, no command output, no test/build result), terminate as `AGENT OVERTHINKING`.
- If evidence appears regularly and outputs are improving, allow continuation even if total request count is high.

**Consultation fallback gate (required for orchestrators):**
- If expert consultation fails (`timeout` or `error`) and work continues, the orchestrator must emit one explicit fallback line before delegation:
  `Proceeding without [agent] input due to [timeout|error]. Assumption: [one sentence].`
- If this line is missing, terminate the run as `AGENT OVERTHINKING` and re-brief with explicit assumptions.

### Status tokens

| Token | Meaning |
|---|---|
| `GO` | Requirements clear; proceed to next step |
| `NO-GO` | Requirements blocked; numbered blockers follow |
| `APPROVED` | Output accepted; gate passed |
| `FINDINGS` | Output accepted with issues; numbered findings follow |
| `HUNG BUILD` | Long-running process exceeded timeout; killed process; cannot continue |
| `CONCERN` | Agent pushing back on the brief itself |
| `COMPLETE` | Work finished and verified |
| `READY` | Artefact produced and ready for next step |
| `GAP` | Cannot complete without an upstream contract change |

- The status token must be the **first word of the response**.
- Findings and blockers must be **numbered**.
- Each finding includes: location, what is wrong, what risk it creates, suggested direction.
- Custom status tokens are allowed and must be declared in the Delegation Contract with their meaning.

---

## Raising Concerns section

```markdown
## Raising Concerns

> **Concern:** [what the problem is]
> **Risk:** [what goes wrong if we proceed]
> **Recommendation:** [what you need to proceed, or what you would do instead]

State it once, clearly. The architect decides.
```

After stating the concern, the agent stops and returns `CONCERN`. It does not produce partial output.

---

## What Good Looks Like section

Two to four concrete, observable sentences. Not aspirational — verifiable.

---

## Skills section

List only skills the agent actively uses. Do not add skills defensively. Skills are resolved at invocation time. Listing a skill the agent does not need wastes context budget.

---

## Explore briefs — batch facts to minimize API calls

The `#explore` agent is read-only and safe to parallelize. When the orchestrator invokes it multiple times sequentially, each invocation costs an API call and rounds of context loading.

**Principle:** Ask for all facts needed in one call. Name exact files or glob patterns, ask for at most 3–5 facts per call. If you need more, you can split into multiple calls, but run those calls in parallel — not sequentially.

**Good:**
```
Agent:    #explore
Context:  Textoria game setup sequence
Input:    Read these 4 files and report:
          1. GameSetupService line where World is initialized
          2. IGameSetupService interface contract
          3. Program.cs where SetupIfNewGameAsync is called
          4. MenuCommandHandler where narrative is displayed
```

**Bad:**
```
Agent: #explore
Input: Read GameSetupService

[result]

Agent: #explore
Input: Read Program.cs

[result]

Agent: #explore
Input: Read MenuCommandHandler
```

The second pattern wastes two API calls that could have been one. The orchestrator should batch explore facts naturally during planning — if you think you need to invoke `#explore` twice in a row, you should instead combine them into one brief.

---

## Process and timeout management

If your agent invokes long-running processes (builds, tests, deployments), define an explicit timeout in your role section and declare it in your Delegation Contract.

**Timeout discipline:**
- Set a reasonable time limit based on typical execution (e.g., 15 minutes for a full test suite).
- If the process exceeds the timeout, kill it immediately. Do not retry.
- Return a custom status token (e.g., `HUNG BUILD`) that signals the orchestrator to investigate root cause.
- Include the timeout value in the token name or in the status details so the orchestrator knows why the process was killed.

**Why:** A hung process blocks the pipeline and causes the agent to retry indefinitely. Killing cleanly and reporting the failure lets the orchestrator decide on root cause investigation instead of blindly retrying.

---

## File size and structure discipline (required)

**Target size: ≤100 lines.** Instructions bloat → overthinking loops.

| Size | Action |
|---|---|
| ≤100 lines | Ideal. Agent reads whole file in one pass. |
| 101–150 lines | Acceptable. Split at next natural boundary. |
| > 150 lines | Mandatory split. Extract by responsibility. |

**One-pass protocols beat complex decision trees:** Avoid nested branches, overlapping responsibilities, multiple phases with overlapping gates. Write protocols as numbered steps, not flowcharts. Example: "1. Read file. 2. Check. 3. Return verdict." — not "If X then read, else explore, except when Y...".

**Mandatory response template:** Every completion status must have a format example shown in the agent instructions:
```
STATUS: TOKEN
KEY_FIELD: value
KEY_FIELD: value
```
Agents follow templates without interpretation. No more prose parsing.

## Model-friendly writing style (required)

Write instructions so small/fast models can execute with minimal interpretation.

- Prefer short imperative sentences: `Edit`, `Run`, `Verify`, `Return`.
- Use explicit limits: counts, timeouts, line ranges, and request caps.
- Avoid abstract words like `thorough`, `comprehensive`, `deep`.
- Provide a fixed response template for each completion status.
- Keep task briefs to one objective and one done condition.

**Good brief shape:**
1. Objective
2. Exact files or patterns
3. Exact commands or operations
4. Done condition (what success looks like)
5. Expected status token(s)

**Multi-feature rule (required):**
- Do not bundle multiple features in one write-agent brief.
- Split into sequential micro-briefs: one feature, one done condition, one verification checkpoint.
- Merge outcomes only after each feature returns evidence-backed completion.

**Early verification checkpoint (required for write agents):**
- Briefs must include: "Run first build/test/verify command within first 2–3 requests."
- If no verification evidence appears by request 3, return `CONCERN` or terminate as `AGENT OVERTHINKING`.

**Background consultation pattern (for orchestrators):**
- If you need expert input (balance, design, domain) that a write agent will depend on, do NOT use background mode.
- Use foreground blocking calls so you receive results before delegating to the write agent.
- If consultation fails (timeout or error) and you still proceed, add one explicit line before delegation: "Proceeding without [agent] input due to [timeout|error]. Assumption: [one sentence]."
- Never delegate to developer with hardcoded domain values while a consultation is still running.
- Never delegate to a write agent after consultation failure without an explicit fallback assumption.

---

## Common mistakes

**Description too vague.** "Does AI stuff" → orchestrator routes everything here. Be specific about domain and output format.

**No status token.** Returning prose without a leading token means the orchestrator must parse natural language to gate. It will guess wrong. Always return a token first.

**Partial output on CONCERN.** Push back cleanly — either produce the full output or raise the concern, not both.

**Writing outside your layer.** Stop at the boundary and raise a `GAP`. Do not overstep into another agent's domain.

**Findings without numbers.** The orchestrator routes findings back by number. Unnumbered findings cannot be tracked. Always number findings.

**Incomplete initial brief.** If a write agent invokes a read agent (fact-gathering) after starting work, it signals the initial brief was incomplete. This is an orchestrator failure. The architect must gather all upstream facts before invoking the write agent. The architect's gates should reject this pattern — if a write agent calls for context mid-work, re-gather facts and re-invoke with a complete brief.

**Contradictory briefs.** Do not ask an agent to violate its stated constraints. If a brief creates a conflict between requirements and agent design, architect must rewrite the brief or choose a different agent. Contradictory briefs cause confusion, constraint violations, and overthinking loops.

**Overthinking before writing (write agents).** Write agents should produce output first (test, file, sketch), run/verify it immediately, then iterate. Do not spend 2+ minutes planning or designing before taking action. Tight feedback loops (produce → verify → observe → refine) beat careful deliberation. Planning slows down the pipeline; fast feedback reveals problems immediately. If you have more than 2–3 API requests before running your first build/test/verification command, you are overthinking — execute now.

**High request count with real progress.** A high request count is acceptable only when the agent shows repeated evidence (file edits + build/test output) at a steady cadence. Treat this as healthy iteration, not overthinking.

**High request count without evidence.** If requests stack up and there is no new command output or file change evidence, that is overthinking even when the brief is complex. Terminate and re-brief with a smaller first step.

**Bundled feature briefs.** "Implement A, B, C" in one write-agent run usually causes planning churn and delayed execution. Split into A → B → C with independent verification after each.

**Context hunting after starting work.** If a write agent spends time reading code "to understand the context better" instead of starting work, it signals either (1) incomplete brief or (2) agent overthinking. Both are failures. Either raise a `GAP` for incomplete brief, or jump into production (test, file, implementation) and let verification results guide refinement.

**Reading entire files.** Read only what the question requires. `glob` to find candidate files, `grep` to locate the specific symbol, then read only the 20–40 lines around each hit. A full-file read of anything over 100 lines signals the scope is too wide. Browsing for "context" wastes API context budget. Name the exact lines you need and read only those.

**No parallelization declaration.** Every agent must state in its Role section whether it is safe to parallelize. Omitting this forces the orchestrator to guess — and it will guess wrong, either serializing work that could run in parallel, or running write agents concurrently and causing API saturation.

**Role section without a scope ceiling.** Describing what an agent does without stating what it does not do is incomplete. An unbounded role drifts. State the boundary explicitly, even when it seems obvious.

**Retrying indefinitely on hung processes.** If a long-running operation (build, test, deploy) exceeds its timeout, kill the process and report the failure with a custom status token. Do not retry silently — the orchestrator must investigate root cause. A hung process in a retry loop will exhaust the API and block the pipeline.

**Mid-work delegation.** Write agents must not invoke other agents during their work phase to fill gaps in context or requirements. If a write agent needs facts, context, or code samples to complete its task, that signals the orchestrator's input was incomplete. The write agent must raise a `GAP` or `CONCERN` and stop — it does not hunt or explore mid-work. All fact-gathering happens before delegation via `#explore`. The orchestrator is accountable for complete briefs.

**Simplicity as a gate.** Agents with >150 lines of instructions cause overthinking loops. If agent behavior shows >2–3 API requests before any action (code written, test run, file created, command executed), audit the agent's instructions immediately. The problem is usually instruction complexity, contradictory flows, duplicated decision points, or verbose role descriptions. Solution: compress to ≤100 lines. Extract specialist responsibilities into separate agents. One protocol per phase. Fixed response templates. Explicit request limits. When agents are simple, they execute fast.

---

## Example: Compression pattern

**Bad (110 lines, causes overthinking):**
```markdown
## How You Work

If you are writing tests, first understand the requirements by reading the spec...
Then check if similar tests exist in the codebase... You might need to explore...
When you're ready, consider the testing framework... Some projects use X, others Y...
Compare patterns across the test suite... Once you decide, begin...
```

**Good (25 lines, direct execution):**
```markdown
## Core Protocol

1. Read requirements.
2. Write test file with [framework] following project conventions.
3. First test fails (red) — expected.
4. Return `READY`.

Rules: Max 3 API requests total. Never gather facts mid-work.
```

The second agent produces output immediately. The first agent overthinks.
