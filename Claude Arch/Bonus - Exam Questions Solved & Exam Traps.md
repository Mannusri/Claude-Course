# Bonus: Exam Questions Solved and Exam Traps

## Purpose

This bonus lesson is a practical exam playbook for applying the concepts from Episodes 1 through 20 to scenario-based questions.

The exam is not primarily a vocabulary test. Questions describe a system, failure, constraint, or desired outcome and ask for the most reliable architectural response.

## Exam Mindset

For every question:

1. Read the setup and constraints carefully.
2. Identify the symptom.
3. Identify the root cause.
4. Identify the requested property: reliable, guaranteed, cheap, immediate, shared, isolated, or recoverable.
5. Eliminate answers that solve a different problem.
6. Choose the smallest effective fix that satisfies the requirement.

```text
Scenario -> Constraint -> Root cause -> Smallest reliable fix
```

## Domain Priorities

The transcript groups the exam into five broad domains. Weightings and official scope can change, so verify the current Anthropic exam guide before taking the exam.

The practical priority order from the course is:

1. Agentic architecture and orchestration
2. Claude Code configuration and workflows
3. Prompt engineering and structured output
4. Tool design and MCP integration
5. Context management and reliability

The first, third, and fourth priority areas represent the largest combined preparation block in the course material. Do not skip the remaining domains; use this order only when review time is limited.

## Four Major Distractor Traps

### Trap 1: Prompt Fix for a Guarantee

Scenario language:

- Must never happen
- Guaranteed
- Policy requires
- Cannot be overridden
- Must be blocked

Distractor:

- Use stronger wording.
- Add a clearer system prompt.
- Add few-shot examples.
- Ask the model to be more careful.

Correct direction:

- Pre-tool hook
- Prerequisite gate
- Application validation
- Deterministic policy enforcement

Prompts are probabilistic. A guarantee requires code or a hard gate.

### Trap 2: Over-Engineered First Fix

When a question asks for the **most effective first step**, choose the smallest fix that directly addresses the symptom.

Do not immediately choose:

- A new routing service
- A new classifier model
- A new sub-agent hierarchy
- A complete architectural rewrite
- A new microservice

If the issue is unclear tool routing, improve tool names and descriptions first. If the issue is context scope, fix configuration scope first. If a rule must be guaranteed, add the relevant hook first.

### Trap 3: Premature Infrastructure

Do not build infrastructure before optimizing an existing design.

Examples:

- Fix tool descriptions before creating a routing classifier.
- Scope tools before creating more agents.
- Move shared commands into project scope before adding distribution infrastructure.
- Use a prerequisite gate before redesigning the entire workflow.

### Trap 4: Sentiment or Confidence as a Decision Signal

Do not use:

- Customer anger alone
- Task complexity alone
- Model self-reported confidence
- A confidence percentage without calibration

as automatic escalation or authorization signals.

Use explicit observable criteria:

- Customer explicitly requests a human.
- No policy covers the case.
- The agent is unauthorized.
- Bounded recovery attempts are exhausted.
- A required validation fails.
- A source conflict remains unresolved.

## One-Line Filter

When two answers seem plausible, ask:

```text
Does this guarantee the outcome, or merely make it more likely?
```

If the scenario says “must,” “never,” or “guarantee,” choose the deterministic solution.

## Domain 1: Agentic Architecture Drills

### Question 1: Refund Ordering

A customer-support agent has `get_customer`, `lookup_order`, and `process_refund`. The system prompt says to verify identity first, but the agent occasionally calls `process_refund` early.

**Best answer:** Add a `PreToolUse` prerequisite gate that blocks `lookup_order` or `process_refund` until verified customer state is true.

**Why:** A prompt does not guarantee ordering. The gate enforces it in code.

**Reject:** Stronger wording, few-shot examples, or `tool_choice: any`. Those may influence behavior but do not enforce the dependency.

### Question 2: Narrow Decomposition

A research coordinator assigns only battery chemistry, market share, and pricing. The final report omits regulation and supply-chain risk, although source documents discuss them.

**Best answer:** Broaden the initial task decomposition to include regulatory policy and supply-chain risk.

**Why:** A topic that was never assigned cannot be repaired by increasing context or changing synthesis alone.

**Reject:** Larger sub-agent context or direct sub-agent communication. Communication should route through the coordinator.

### Question 3: Parallel Spawning

A coordinator needs three independent research sub-agents but calls the task tool once, waits, then calls it again.

**Best answer:** Emit multiple task tool calls in one coordinator response.

**Why:** One response with multiple independent calls enables parallel execution. Separate turns create sequential latency.

## Domain 2: Tool Design and MCP Drills

### Question 4: Poor Tool Routing

A support agent frequently escalates resolvable requests and sometimes calls refund tools before verification. Every tool has only a one-sentence description.

**Best answer:** Expand descriptions with purpose, invocation criteria, prerequisites, output, negative boundaries, and when not to use the tool.

**Why:** The first issue to fix is ambiguous tool metadata.

**Reject:** A new routing classifier, extra sub-agent, or several few-shot examples as the first step.

### Question 5: Synthesis Tool Scope

A synthesis agent should combine gathered findings but has access to web search, document analysis, database lookup, and `verify_fact`. It begins searching for new information instead of synthesizing.

**Best answer:** Scope its tool access to the relevant tool, such as `verify_fact`, and remove unrelated search tools.

**Why:** Fewer role-specific tools improve routing reliability and reduce unnecessary actions.

### Question 6: Shared MCP Configuration

A project works on one developer's machine, but teammates cannot discover its required MCP servers after cloning.

**Best answer:** Put shared server definitions in the project-root `.mcp.json` and commit the file. Keep credentials in environment variables or a secret manager.

## Domain 3: Claude Code Configuration Drills

### Question 7: Team Slash Command

A custom `/review` command must be available to every developer immediately after cloning the repository.

**Best answer:** Store it in the project's `.claude/commands/` directory and commit it.

**Reject:** A user-level home command, because it is not automatically shared.

### Question 8: Large Migration

A 40,000-line monolith with unclear service boundaries must be migrated to three microservices.

**Best answer:** Use Plan Mode first to inspect dependencies, compare approaches, and approve a phased strategy before execution.

**Reject:** Direct execution, guessing boundaries, or spawning implementation agents immediately.

### Question 9: Conditional Rules

Terraform rules should apply only under `terraform/`, while TypeScript test rules should apply to matching test files across the repository.

**Best answer:** Use `.claude/rules/` files with YAML front matter and `paths` glob patterns.

**Reject:** Putting all rules in a broad `CLAUDE.md` or using `@import` alone when conditional file targeting is required.

### Question 10: CI Hang

Claude works interactively but a GitHub Actions review hangs until the job times out.

**Best answer:** Run Claude with `-p` or `--print` and provide a self-contained non-interactive prompt.

**Reject:** Fabricated flags such as `--headless`, `--batch`, or `--ci`.

## Domain 4: Prompt Engineering and Structured Output Drills

### Question 11: Batch API in a Blocking Check

A team wants to use the message batches API for a pre-merge check that blocks pull requests until review feedback arrives.

**Best answer:** Use standard messages for the blocking review. Batches are asynchronous and may take up to the supported processing window.

**Good batch use:** Nightly test generation, offline audits, or independent document processing.

### Question 12: Multi-Pass Review

A single agent reviews a 14-file pull request inconsistently and misses cross-file breaks.

**Best answer:** Run independent per-file reviews, then a separate cross-file integration pass.

**Why:** Per-file passes provide depth; the integration pass provides breadth.

### Question 13: Structured Extraction

A JSON-schema extraction pipeline returns syntactically valid data, but the due date is fabricated because it is absent from the source.

**Best answer:** Make the field required but nullable, return `null` when absent, and route the record for review if required.

**Why:** Retries correct format errors, not missing facts.

### Question 14: False Positives

A reviewer reports dozens of style and naming issues. Developers ignore all findings, including security issues.

**Best answer:** Replace vague confidence instructions with explicit categorical include and skip rules. Temporarily disable a persistently noisy category while improving it.

**Reject:** “Be more conservative,” lower confidence thresholds, or add infrastructure before clarifying the rules.

## Domain 5: Context and Reliability Drills

### Question 15: Model Confidence Escalation

An agent escalates whenever its self-reported confidence drops below 70%, but it still misses complex disputes and escalates simple ones.

**Best answer:** Replace confidence-based routing with explicit escalation criteria: human request, policy gap, authorization failure, unresolved conflict, or exhausted recovery attempts.

### Question 16: Empty Result Semantics

A document agent returns the same empty result when a source is unavailable and when a successful search finds no matches.

**Best answer:** Return structured error context distinguishing access failure from valid empty results, including retryability, attempted query, and partial results.

### Question 17: Human Review for Extraction

A system has 97% overall accuracy but 60% accuracy on handwritten receipts.

**Best answer:** Use stratified sampling and category-specific metrics, then route the failing category to human review.

### Question 18: Source Conflict

Two sources report different market sizes.

**Best answer:** Preserve both claims and source mappings, mark the conflict unresolved, and escalate for coordinator or human review.

**Reject:** Choosing the highest, choosing the lowest, or averaging the values.

## Bonus Domain: Built-in Tool Navigation

### Question 19: Find a Function Call

Find every call to `process_refund` across a repository.

**Best answer:** Use `grep`, because the request is about file content.

### Question 20: Find Files by Pattern

Find all `*.test.tsx` files under `src`.

**Best answer:** Use `glob`, because the request is about paths and filenames.

### Question 21: Ambiguous Edit

An `edit` operation fails because the old string appears several times.

**Best answer:** Read the full file, make the intended change, and write the complete updated file.

## Fast Elimination Table

| Scenario signal | Eliminate | Prefer |
| --- | --- | --- |
| Must, never, guarantee | Prompt-only fixes | Hook or application gate |
| Wrong tool selected | New infrastructure | Better name, description, and scope |
| Team-wide command | User home configuration | Project `.claude/commands/` |
| Large architecture change | Direct execution | Plan Mode |
| CI waiting for result | Batch API | Standard messages |
| Nightly independent work | Blocking synchronous design | Message batches |
| Missing source field | More retries or inference | Nullable field and review |
| Self-review bias | Same session twice | Independent session |
| Overall metric hides failures | Global average only | Stratified sampling |
| Low-confidence field | Silent discard or force-through | Human review |
| Conflicting sources | Average or choose one | Preserve both and escalate |
| Content search | `glob` | `grep` |
| Path search | `grep` | `glob` |

## Exam-Day Playbook

### Read the Setup Twice

Identify:

- What is failing?
- Who is waiting?
- Is the task blocking?
- Is the requirement probabilistic or guaranteed?
- Is the issue local or architectural?
- Is context missing, malformed, stale, or semantically wrong?

### Trace Symptom to Root Cause

Do not jump directly to the most elaborate answer.

```text
Symptom -> Root cause -> Smallest effective fix
```

Examples:

- Wrong tool -> Description or scope problem
- Unauthorized action -> Pre-tool gate
- CI hang -> Non-interactive `--print`
- Duplicate review comments -> Prior findings
- Wrong structure -> Schema or tool-use enforcement
- Wrong value in valid structure -> Semantic validation
- Context exhaustion -> Scratchpad, explore sub-agent, or compact strategy

### Watch for Fabricated Features

Options may contain plausible-looking flags, fields, or capabilities that do not exist. Treat made-up environment variables, CLI flags, and parameters as distractors unless supported by the course or official documentation.

### Flag Hard Questions

Do not allow one difficult question to consume the time needed for several easy questions. Mark it, continue, and return later.

## Final Cheat Sheet

- Guarantee beats probability.
- Hooks beat prompts for non-negotiable rules.
- Smallest effective fix beats premature infrastructure.
- Tool descriptions and scope come before routing infrastructure.
- Project scope means shared; user scope means personal.
- Plan Mode precedes large or irreversible changes.
- `-p` / `--print` enables unattended CI execution.
- Batch API is for non-blocking asynchronous work.
- `custom_id` correlates batch requests and results.
- `grep` searches contents; `glob` searches paths.
- `edit` needs a unique anchor; otherwise read and write.
- Few-shot examples should be targeted and reasoning-backed.
- Schemas enforce structure, not truth.
- `null` is better than fabricated data.
- Fresh sessions reduce self-review bias.
- Stratified sampling reveals category failures.
- Field-level confidence beats document-level averages.
- Preserve claim-source mappings and temporal dates.
- Never silently suppress sub-agent errors.
- Never let a worker terminate the entire pipeline without coordinator judgment.
- Human requests, policy gaps, unresolved conflicts, and exhausted recovery justify escalation.

## Final Summary

Scenario questions reward architectural judgment. Identify the constraint, eliminate distractors that only make success more likely, and choose the smallest solution that actually satisfies the requirement.

When the scenario says **must, never, or guarantee**, choose deterministic enforcement. When it describes uncertainty, missing provenance, category-specific failures, or human-risk decisions, preserve evidence and route the case to the appropriate coordinator or human reviewer.
