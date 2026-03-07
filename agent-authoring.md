# Agent Authoring

Skill instruction for writing GitHub Copilot agents — sub-agents and orchestrators.

---

## I. Shared Foundations

### File structure

Agents live in `.github/agents/` as `.agent.md` files. YAML frontmatter + markdown body.

- `name` — lowercase, hyphenated, matches filename. Used as `#my-agent` handle.
- `description` — specific domain + output format. Vague descriptions cause misrouting.
- `tools` — only what the agent needs. Tools define the boundary — if an agent has a tool, it will use it. No defensive additions.
- `model` — optional. Typical tiers: **Mechanical** (free — file reading, builds, trivial edits), **Reasoning** (free — design judgment, creative input), **Premium** (paid — complex implementation). Specific model IDs belong in the orchestrator's routing table only.

### Body sections

Default order: `Role` → `How You Work` → `Delegation Contract` → `Raising Concerns` → `What Good Looks Like` → `Skills`

Reorder when it improves clarity. Domain-specific sections (e.g., "Running Tests", "Test Categories") go after standard sections. Scouts may use minimal structure (Role → Reading Discipline → Output Format → Hard Rules).

### Status tokens

| Token | Meaning |
|---|---|
| `GO` / `NO-GO` | Requirements clear / blocked (numbered blockers) |
| `APPROVED` / `FINDINGS` | Accepted / accepted with numbered issues |
| `COMPLETE` / `READY` | Work verified / artifact produced |
| `CONCERN` | Pushing back on brief |
| `GAP` | Cannot complete without upstream change |
| `LOST` | Cannot determine next action |
| `HUNG BUILD` | Process exceeded timeout |

Token is always the **first word** of the response. Each agent adds domain-specific tokens in its Delegation Contract (e.g., `TESTS WRITTEN`, `SPEC READY`, `AUDIT COMPLETE`). Same rules: first word, uppercase, unambiguous.

### Writing style

Short imperative sentences. Explicit limits (counts, timeouts, line ranges). No abstract words (`thorough`, `comprehensive`). Fixed response templates per status. Numbered steps, not flowcharts.

### File size

**Sub-agents:** ≤100 lines ideal, ≤150 max. **Orchestrators:** 200–400 lines. Every instruction line is a decision the agent weighs — compress where possible.

### Infra resilience

Transport errors (`Request was aborted`, `CAPIError`) are infra failures. Every agent handles them identically:

1. **2-retry limit.** Same tool/model call fails twice → stop.
2. **Return `CONCERN`** naming: (a) exact files touched/unprocessed, (b) current state, (c) next smallest step.
3. **No silent retries.** Never retry a third time.

Include a fail-fast sentence in `How You Work`.

---

## II. Agent Archetypes

Pick the archetype first — it determines tools, parallelization, and delegation contract shape.

| Archetype | Purpose | Parallel? | Standard tools |
|---|---|---|---|
| **Orchestrator** | Plans, classifies, delegates. Never writes code/docs. | N/A | `task`, `read_agent`, `list_agents`, `store_memory`, `report_intent`, `ask_user`, `todo` |
| **Scout** | Read-only fact-finding. Structured findings. | Yes | `view`, `grep`, `glob` |
| **Implementer** | Writes/edits source code. Runs builds/tests. | No | `view`, `edit`, `create`, `grep`, `glob`, `get_errors`, `runTests`, `store_memory` |
| **Tester** | Writes tests, enforces quality gates. | No | `view`, `edit`, `create`, `powershell`, `grep`, `glob`, `get_errors`, `runTests`, `store_memory` |
| **Reviewer** | Read-only code review. APPROVED or FINDINGS. | Yes | `view`, `grep`, `glob`, `get_errors` |
| **Advisor** | Read-only domain expert. Briefs/specs/creative input. | Yes | `view`, `grep`, `glob` |
| **Documentarian** | Writes docs only — never source code. | No | `view`, `edit`, `create`, `grep`, `glob`, `store_memory` |
| **Auditor** | Read-only analysis against standards. Findings reports. | Yes | `view`, `grep`, `glob` |

**Tool rule:** Read-only archetypes never get `edit`, `create`, or `powershell`. Add `powershell` to write archetypes only when they need shell commands directly.

**Split rule:** Same archetype + different domain → separate agents. Same domain + different archetype → separate agents. Needs tools from two archetypes → split.

### Archetype profiles

#### Scout
**Tokens:** Findings structure only (no prefix token). **Contract:** Query → Findings (topic, path:line, fact, gaps). **Pattern:** `glob` → `grep` → `view` with line range. One pass + one follow-up max. **Ceiling:** Return findings only. Never write or edit. **Structure:** Minimal — Role → Reading Discipline → Output Format → Hard Rules.

#### Implementer
**Tokens:** `IMPLEMENTATION COMPLETE`, `TESTS WRITTEN`, `TESTS GREEN`, `CONCERN`, `GAP`. **Contract:** Implement (feature → code + green tests) · Fix findings (findings → fixed code) · Bug fix (failing test → `TESTS WRITTEN`, fix → `TESTS GREEN`). **Pattern:** `grep`/`glob` to locate, read 50–80 lines around hit, then write. Test-driven: red → green → refactor. **Ceiling:** No documentation, no design decisions. `CONCERN` if brief requires judgment. **Infra fallback:** files touched + build/test state + next step.

#### Tester
**Tokens:** `GO`, `NO-GO`, `APPROVED`, `FINDINGS`, `PASS`, `BLOCK`. **Contract:** Requirements review (→ GO/NO-GO) · Write tests (→ failing tests) · Review tests (→ APPROVED/FINDINGS) · Gate check (→ PASS/BLOCK). **Pattern:** Requirements first — challenge ambiguity. Read production class before writing. One risk area per call. **Ceiling:** No production code. No fixing code to pass tests. **Infra fallback:** test project + filter + next step.

#### Reviewer
**Tokens:** `APPROVED`, `FINDINGS`, `CONCERN`. **Contract:** Review (full diff → APPROVED/FINDINGS) · Second-pass (fixed diff → APPROVED/remaining). **Pattern:** Verdict first. Full diff in one call. Line ranges for files >150 lines. **Ceiling:** No rewriting code — raise findings, suggest direction. **Infra fallback:** unreviewed files + next slice.

#### Advisor
**Tokens:** `BRIEF READY`, `SPEC READY`, `CONSISTENT`, `ISSUES`. **Contract:** Design (intent → brief/spec) · Review (→ CONSISTENT/ISSUES) · Resolve GAP (→ revised brief). **Pattern:** Design intent in one sentence first. Structured dimensions. Every spec includes a Design Intent line. **Ceiling:** No code. `CONCERN` if constraints make design impossible. **Infra fallback:** partial brief + remaining sections.

#### Documentarian
**Tokens:** `DONE`, `CONCERN`. **Contract:** Create (context → doc) · Update (change → section) · Technical note (topic → doc ≤150 lines). **Pattern:** Present tense, active voice, short sentences. Tables over bullets. Mermaid for flows >2 parts. **Ceiling:** No source code. Only decisions with lasting consequences. **Infra fallback:** files written + files remaining.

#### Auditor
**Tokens:** `AUDIT COMPLETE`, `NO FINDINGS`. **Contract:** Full audit (→ findings) · Targeted audit (→ scoped findings) · Gap check (→ gaps only). **Pattern:** Inventory → Gap detection → Accuracy check → Report. Severity: HIGH/MED/LOW. **Ceiling:** Do not fix findings — return and let orchestrator delegate. **Infra fallback:** completed sections + blocked sections.

---

## III. Writing Sub-Agents

### Role section

One paragraph: (1) domain owned, (2) what it produces, (3) what it does **not** do. Scope ceiling is mandatory.

**Parallelization declaration (required):**
- Read-only: *"Analysis-only. Safe to parallelize."*
- Write: *"Write agent. Run sequentially — never in parallel with another write agent."*

### Delegation Contract

```
| Phase | You receive | You return |
|---|---|---|
| Phase name | Input from orchestrator | Output + status token |
```

### Runtime behaviors (in How You Work)

**Objective anchor** (recommended for implementers/testers): State `OBJECTIVE:` and `DONE WHEN:` before any actions. Can't state both in one sentence each → `CONCERN`.

**Breadcrumbs** (recommended for multi-step agents): End each response with `DID:` and `NEXT:`. Can't state NEXT in one sentence → `LOST`. Lets orchestrator detect drift by comparing NEXT to OBJECTIVE.

### Raising Concerns

```
> **Concern:** [problem]  **Risk:** [consequence]  **Recommendation:** [what you need]
```

Return `CONCERN` and stop. No partial output.

### Brief validation

**Orchestrator** validates five fields before delegating: objective, input files, constraints, done condition, expected token. **Sub-agents** validate domain-specific prerequisites only. Missing something critical → `GAP` naming the item.

### Execution rules (tripwires)

An agent that hits a tripwire emits the token and stops — no deliberation.

| # | Rule | Tripwire |
|---|---|---|
| 1 | **Produce first.** No extended planning. | Output within 3 requests or `GAP`/`LOST`. |
| 2 | **No mid-work exploration.** Facts arrive in brief. | Exploration after brief → `GAP`. |
| 3 | **No self-rescue.** | Second read of same region → `LOST`. |
| 4 | **Verify often.** | 4 consecutive requests without verify → force verify. |
| 5 | **Read surgically.** `grep` → 20–40 lines. | Max 3 reads before first output. No full-file reads >100 lines. |
| 6 | **Timeouts.** | Exceeded → kill → `HUNG BUILD`. |
| 7 | **No monologue.** | >2 sentences reasoning before tool call → `LOST`. |
| 8 | **Fail fast on infra.** | 2 transport failures → `CONCERN` (see §I). |
| 9 | **Bounded execution.** | One pass + one verify per invocation. |

### What Good Looks Like / Skills

- `What Good Looks Like`: 2–4 verifiable sentences. "All tests pass" — good. "Code is clean" — bad.
- `Skills`: Only skills actively used.

---

## IV. Writing Orchestrators

The orchestrator plans, adapts, and makes judgment calls. These are guidelines, not rigid gates.

### Required sections

- **Cannot Do list** — top of file. Hard limits: no writing code, no running builds, no briefing without model routing.
- **Execution Loop** — Classify → Explore → Plan → Execute → Gate → Verify → Done.
- **Task Classification** — table: task signals → pipeline weight (FULL/FAST/TRIVIAL).
- **Pipelines** — table per weight: step, agent, input, gate.
- **Agent Selection** — table: task type → agent. "If nothing fits" → general implementer.
- **Model Routing** — table: agent + task → model ID. Pass `model:` in every brief.
- **Speed Defaults** — one implementation pass + one verification + one reviewer pass.
- **Reliability Guardrails** — fallback routing for infra failures.

### Brief completeness

Five fields: (1) objective, (2) input files, (3) constraints, (4) done condition, (5) expected token.

Self-check: can the agent produce within 1–2 requests? If not, narrow or add files. Anti-patterns: unnamed files, missing locations, compound objectives ("A and B" → split).

### Monitoring sub-agents

| Signal | Indicator | Response |
|---|---|---|
| Drift | NEXT diverges from OBJECTIVE | Re-brief narrower |
| Read spiral | >3 reads without output | Intervene |
| Planning loop | >2 responses of analysis | Intervene |
| Scope creep | Files outside domain | Stop |
| Infra failure | `CONCERN` citing transport errors | Re-route (see Fallback routing) |
| Stall | Same region edited >2× without verify | Force verify or stop |

### Fallback routing

When agent returns `CONCERN` from infra failure:

1. Do not re-invoke same agent with same brief.
2. Re-route to different agent with narrower, file-scoped brief.
3. One diagnose+fix attempt max, then GO/NO-GO.
4. Keep fallback briefs compact.

### Recovery

Stop agent. Diagnose: incomplete brief (yours) or complex design (agent's)? Gather facts → re-brief, or simplify → re-invoke. Never retry unchanged.

### Consultation

Foreground blocking calls for expert input. If consultation fails, state assumption before delegating. Never delegate with hardcoded values while consultation runs.

### Orchestrator freedom

The orchestrator is accountable for **outcomes**, not process. It may adjust limits, reorder steps, allow high request counts when evidence supports it. Did the sub-agent produce verified artifacts matching the objective? Path was acceptable.

---

## V. Common Mistakes

**[sub-agent]** No status token → orchestrator must parse prose.
**[sub-agent]** Partial output on CONCERN → push back cleanly or produce full output, not both.
**[sub-agent]** Writing outside your layer → `GAP`.
**[sub-agent]** Lost without knowing it → reading files without converging. Most dangerous failure.
**[sub-agent]** Silent infra retry → fail fast after 2 attempts.
**[sub-agent]** Vague infra CONCERN → name files, state, next step.
**[orchestrator]** Vague description → misrouting.
**[orchestrator]** Incomplete briefs → #1 cause of drift.
**[orchestrator]** Bundled features → split into sequential micro-briefs.
**[orchestrator]** Re-invoking after infra CONCERN → re-route instead.
**[both]** No parallelization declaration.
**[both]** Role without scope ceiling.
**[both]** No fail-fast clause in How You Work.
