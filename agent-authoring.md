# Agent Authoring

Skill instruction for writing GitHub Copilot agents — sub-agents under an autopilot orchestrator, and orchestrators themselves.

---

## I. Shared Foundations

### File structure

Agents live in `.github/agents/` as `.agent.md` files. YAML frontmatter + markdown body.

- `name` — lowercase, hyphenated, matches filename. Used as `#my-agent` handle.
- `description` — specific domain + output format. Vague descriptions cause misrouting.
- `tools` — only what the agent needs. No defensive additions. Names must match exact runtime tool identifiers. List only tools the agent calls directly.

### Body sections (in order)

`Role` → `How You Work` → `Delegation Contract` → `Raising Concerns` → `What Good Looks Like` → `Skills`. No custom sections.

### Status tokens

| Token | Meaning |
|---|---|
| `GO` | Requirements clear; proceed |
| `NO-GO` | Blocked; numbered blockers follow |
| `APPROVED` | Output accepted; gate passed |
| `FINDINGS` | Accepted with issues; numbered findings follow |
| `HUNG BUILD` | Process exceeded timeout; killed |
| `CONCERN` | Pushing back on the brief |
| `COMPLETE` | Work finished and verified |
| `READY` | Artifact produced, ready for next step |
| `GAP` | Cannot complete without upstream change |
| `LOST` | Cannot determine next action; returning control |

Token is always the **first word** of the response. Findings/blockers are **numbered**. Custom tokens must be declared in the Delegation Contract.

### Writing style

- Short imperative sentences: `Edit`, `Run`, `Verify`, `Return`.
- Explicit limits (counts, timeouts, line ranges). No abstract words (`thorough`, `comprehensive`).
- Fixed response templates per status. One objective, one done condition per brief.
- Numbered steps, not flowcharts. One-pass protocols beat decision trees.

### File size

≤100 lines ideal. 101–150 acceptable. >150 mandatory split. Bloat causes overthinking. Every instruction line is a decision the agent weighs before acting.

---

## II. Writing Sub-Agents

### Objective anchor (required)

First thing the agent writes after receiving a brief — before any reads or actions. This is a runtime behavior — include it in the agent's `How You Work` section so the agent states it at the start of every invocation.

```
OBJECTIVE: [one sentence — what I will produce]
DONE WHEN: [one sentence — exit condition]
```

Cannot state both in one sentence each? Brief is too broad → return `CONCERN`.

### Role section

One paragraph answering: (1) what domain it owns, (2) what it produces, (3) what it does **not** do. The scope ceiling is mandatory — unbounded roles drift.

**Parallelization declaration (required):**
- Read-only agents: *"Analysis-only. Safe to parallelize."*
- Write agents: *"Write agent. Run sequentially — never in parallel with another write agent."*

### Delegation Contract

```
| Phase | You receive | You return |
|---|---|---|
| Phase name | Input from orchestrator | Output + status token |
```

### Breadcrumbs (required)

Every response ends with:

```
DID: [what was just completed — one sentence]
NEXT: [the single next action — one sentence]
```

Cannot state NEXT in one sentence → return `LOST`. Breadcrumbs enable self-correction and let the orchestrator detect drift by comparing NEXT to the OBJECTIVE anchor.

### Raising Concerns

```
> **Concern:** [problem]  **Risk:** [consequence]  **Recommendation:** [what you need]
```

Return `CONCERN` and stop. No partial output.

### Brief validation (required)

Before starting work, confirm the brief contains: (1) objective, (2) input files, (3) constraints including build/test command, (4) done condition, (5) expected token. If any field is missing or ambiguous → `GAP` with the missing field named.

### Sub-agent execution rules

**Principles guide. Tripwires enforce.** Every rule below has a principle (why) and a tripwire (a countable limit). An agent that hits a tripwire does not deliberate — it emits the specified token and stops.

| # | Principle | Tripwire |
|---|---|---|
| 1 | **Produce first, plan later.** Output artifacts then verify. No extended planning. | File write or terminal command within 3 requests. Brief missing info → `GAP`. Brief complete but no output → `LOST`. |
| 2 | **No mid-work context hunting.** All facts arrive in the brief. | Any exploration after receiving the brief → `GAP`. |
| 3 | **No self-rescue.** If stuck, stop. | Second read of same file region → `LOST`. |
| 4 | **Evidence cadence.** Verify early and often. | First verify within 2–3 requests. Then every 3–5. 4 consecutive requests without verify → force verify. |
| 5 | **Read surgically.** `grep` → read 20–40 lines around the hit. | Max 3 reads before first output. Max 2 consecutive reads after. No full-file reads over 100 lines. |
| 6 | **Timeouts.** Declare in Role section. | If exceeded → kill process → `HUNG BUILD`. No retries. |
| 7 | **No internal monologue.** Output is files, results, tokens. | More than 2 sentences of reasoning before a tool call → execute or `LOST`. |

The orchestrator monitors these same signals externally. Tripwires are your self-enforcement; the orchestrator is the safety net.

### What Good Looks Like / Skills

- `What Good Looks Like`: 2–4 verifiable sentences. Not aspirational. Litmus test: could a machine or grep command confirm the sentence? "All tests pass" — verifiable. "Code is clean" — aspirational.
- `Skills`: Only skills actively used. No defensive listing.

---

## III. Writing Orchestrators

The orchestrator has more freedom than sub-agents. It must plan, adapt, and make judgment calls. The constraints below are guidelines that inform good judgment — not rigid gates.

### Brief completeness

Before delegating, ensure the brief contains:
1. **Objective** — one sentence
2. **Input files** — exact paths or globs
3. **Constraints** — limits, framework, language
4. **Done condition** — observable, not aspirational
5. **Expected token(s)**

Self-check: can the agent start producing within 1–2 requests from this brief alone? If not, narrow the objective or add input files.

Drift-causing anti-patterns: unnamed files ("implement the feature"), missing locations ("fix the bug"), compound objectives ("A and B" → split).

### Explore batching

Batch fact-gathering into single `#explore` calls with 3–5 named facts. Parallel calls are fine; sequential calls for the same batch waste API requests.

### Monitoring sub-agents

Watch for drift signals and use judgment on when to intervene:

| Signal | Indicator | Response |
|---|---|---|
| Objective drift | NEXT breadcrumb diverges from OBJECTIVE | Re-brief with narrower scope |
| Read spiral | >3 file reads without output | Likely overthinking — intervene |
| Planning loop | Plans/analysis instead of artifacts for >2 responses | Intervene |
| Scope creep | Files modified outside declared domain | Stop, review boundaries |
| Question cascade | >2 clarifying questions | Brief was probably incomplete — re-gather, re-brief |
| Repeated reads | Same file/region read twice | Agent is stuck — re-brief with guidance |
| Stall | Same file region edited >2 times without verify | Polishing, not converging — force verify or stop |
| No breadcrumb | Missing DID/NEXT footer | Reject response, re-invoke |

Sub-agents have internal tripwires for these signals. External monitoring catches cases where self-enforcement fails.

### Recovery

1. Stop the agent immediately — further requests deepen drift.
2. Diagnose: incomplete brief (your fault) or complex agent design (agent fault)?
3. Incomplete brief → gather facts via `#explore`, re-brief with all five fields.
4. Agent too complex → simplify instructions, narrow role, re-invoke.
5. Never retry the same brief unchanged.

### Consultation

- Use foreground blocking calls for expert input that write agents depend on.
- If consultation fails, state explicitly before delegating: *"Proceeding without [agent] input due to [reason]. Assumption: [one sentence]."*
- Never delegate with hardcoded values while consultation is still running.

### Orchestrator freedom

The orchestrator is accountable for pipeline outcomes, not for following a script. It may:
- Adjust request caps based on task complexity — simple tasks get tight limits, complex tasks get room.
- Reorder, skip, or combine steps when evidence supports it.
- Allow high request counts from sub-agents when evidence (file edits + test output) appears at a steady cadence.
- Make judgment calls on ambiguous signals — not every read spiral is overthinking.

The orchestrator's constraint is **outcomes**: did the sub-agent produce verified artifacts that match the objective? If yes, the path was acceptable regardless of request count.

---

## IV. Common Mistakes

**[sub-agent]** No status token → orchestrator must parse prose. Always lead with a token.
**[sub-agent]** Partial output on CONCERN → push back cleanly or produce full output, not both.
**[sub-agent]** Writing outside your layer → raise `GAP`, don't overstep.
**[sub-agent]** Findings without numbers → unnumbered findings can't be tracked.
**[sub-agent]** Context hunting after starting → raise `GAP` or execute. Don't explore mid-work.
**[sub-agent]** Lost without knowing it → confidently reading files without converging on objective. Most dangerous failure mode.
**[sub-agent]** Self-rescue attempts → reading broadly when stuck. Return `LOST` instead.
**[sub-agent]** Internal monologue as work → multi-paragraph reasoning burns requests without artifacts. Execute or `LOST`.
**[orchestrator]** Vague description → routes everything to one agent. Be domain-specific.
**[orchestrator]** Incomplete briefs → #1 cause of agent drift. Gather facts before delegating.
**[orchestrator]** Bundled features → "Implement A, B, C" causes churn. Split into sequential micro-briefs.
**[orchestrator]** Contradictory briefs → rewrite or choose a different agent. Don't force conflicts.
**[orchestrator]** Retrying hung processes → kill, report, investigate root cause. No silent retries.
**[both]** No parallelization declaration → orchestrator guesses wrong. Always declare.
**[both]** Role without scope ceiling → unbounded roles drift. State boundaries explicitly.
**[both]** >150 lines of instructions → compress. Bloated agents overthink.
