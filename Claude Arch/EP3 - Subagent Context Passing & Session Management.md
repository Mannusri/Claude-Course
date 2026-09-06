# EP3: Subagent Context Passing & Session Management

## Lesson Goal

This lesson explains how to pass reliable context to isolated sub-agents and how to choose the correct session-management strategy when working with Claude-based agentic systems.

The main exam topics are:

- Clean-slate sub-agent context
- Explicit context injection through task prompts
- Structured findings and claim-to-source mappings
- Source attribution preservation
- Conflict annotation and escalation
- Named session resumption
- Fork sessions
- Fresh sessions with hypothesis-based summaries
- Session-management anti-patterns

## The Context Problem

Sub-agents start with a clean slate. They do not automatically inherit the coordinator's conversation history, previous tool results, or the context of other sub-agents.

If a coordinator has already searched documents, called tools, and gathered findings, a newly spawned synthesis agent knows only what is included in its task request.

```text
Coordinator context
    -> Explicitly selected information
    -> Task prompt
    -> Fresh sub-agent context
```

This isolation is intentional. It prevents context contamination and keeps workers focused, but it means that context passing is the coordinator's responsibility.

## The Core Rule

**Every sub-agent starts with a clean slate. Its entire working world is what the coordinator puts in the prompt and task input.**

Never assume that a sub-agent knows:

- What the coordinator discussed earlier
- What another sub-agent discovered
- Which files or sources were already inspected
- The user's original constraints
- The meaning of a vague reference such as “the findings above”

Without explicit context, the sub-agent may hallucinate, produce a generic answer, or solve the wrong problem.

## Context Injection Through the Task Tool

The coordinator should extract relevant information and place it into the sub-agent's task prompt.

### Weak Prompt

```text
Here are the previous research findings. Synthesize them into a final report.
```

This prompt does not include the findings. The sub-agent cannot access the coordinator's hidden context.

### Strong Prompt

```text
Produce a structured research report.

Findings to synthesize:
[
  {
    "claim": "Claude's tool-use stop reason requires the full assistant message before tool results.",
    "source_url": "https://example.com/claude-docs",
    "source_title": "Claude API documentation",
    "retrieved_at": "2026-09-06",
    "confidence": "high"
  }
]

Requirements:
- Cite the source URL for every claim.
- Preserve each retrieved date.
- Flag conflicting findings instead of silently choosing one.
```

The stronger prompt includes the actual data, constraints, attribution requirements, and expected behavior.

## Structured Context Passing

Do not pass important findings as an unstructured prose summary when later agents need to verify, cite, or compare them. Pass structured objects instead.

A useful research finding can include:

- Finding or claim text
- Claim ID
- Source ID or URL
- Source title
- Retrieval timestamp
- Confidence level
- Relevant page, section, or evidence

Example:

```json
{
  "claim_id": "C001",
  "text": "Sub-agents do not inherit the coordinator's conversation history.",
  "source_id": "S001",
  "retrieved_at": "2026-09-06",
  "confidence": "high"
}
```

The next agent should receive the complete structured object, not only the `text` field.

## Claim-to-Source Mapping

Source attribution is easily lost when multiple web results are summarized into prose. Once the source relationship disappears, the final agent cannot reliably determine where a claim came from.

Use explicit claim-to-source mappings:

```json
{
  "claims": [
    {
      "claim_id": "C001",
      "text": "Sub-agents start with isolated context.",
      "source_id": "S001",
      "confidence": "high"
    }
  ],
  "sources": [
    {
      "source_id": "S001",
      "url": "https://example.com/source",
      "title": "Multi-agent architecture guide",
      "retrieved_at": "2026-09-06"
    }
  ]
}
```

### Preservation Rule

When passing data to the next agent, pass both the claims and the sources. Never extract only the claims and discard the source records.

Every synthesis step should preserve or produce claim-source pairs so that the final report remains traceable.

## Handling Conflicting Sources

Different sources may make contradictory claims. A sub-agent should not silently discard one source or decide the final truth without an explicit policy.

### Correct Conflict Workflow

1. Preserve both claims.
2. Preserve the attribution for both sources.
3. Create a conflict object.
4. Mark the resolution state as `unresolved`.
5. Escalate the conflict to the coordinator.
6. Let the coordinator decide how the final result should represent the disagreement.

Example:

```json
{
  "conflict_id": "CONF001",
  "topic": "Parallel sub-agent invocation",
  "claims": [
    {
      "text": "Claim A",
      "source_id": "S001"
    },
    {
      "text": "Claim B",
      "source_id": "S002"
    }
  ],
  "resolution_state": "unresolved",
  "escalate_to": "coordinator"
}
```

### Exam Rule

If a question asks how to handle conflicting sources, choose the answer that **annotates both claims, preserves attribution, marks the conflict unresolved, and escalates it to the coordinator**.

## Session Management

Session management allows an agentic workflow to continue previous work, explore alternatives, or deliberately start over.

The correct strategy depends on whether the previous context is still valid and whether the task is a continuation or an independent exploration.

## Pattern 1: Named Session Resumption

Use a named session when continuing the same task and the existing context remains useful.

Conceptual CLI examples:

```text
claude --session-name research-session
claude --resume research-session
claude --resume
```

The final command represents resuming the most recent session. Verify exact command syntax against the current Claude Code documentation before using it in a live environment.

### Resumption Rule

When resuming, explicitly tell the agent what changed since the previous session.

Example:

```text
Resume the previous analysis. The payments module was refactored and
requirements.txt changed. Re-analyze those files and continue from the
previous findings.
```

Do not assume the agent automatically knows that files were deleted, dependencies changed, or requirements were updated.

### When to Resume

Resume when:

- The same task is continuing.
- Most previous context remains valid.
- Changes are limited and can be clearly described.
- Prior findings are still useful.

## Pattern 2: Fork Session

Use a fork session to explore independent approaches from the same valid baseline.

```text
                       Baseline session
                       /               \
              Fork A: approach A   Fork B: approach B
```

Both branches begin with the baseline context, but subsequent work is independent. Changes in one fork do not alter the other fork or the original baseline.

### When to Fork

Fork when:

- The baseline analysis is valuable and should be preserved.
- You are comparing alternative architectures or implementations.
- You want parallel exploration without mixing the branches.
- The original context is still valid.

Forking is for controlled exploration, not for repairing a heavily stale context.

## Pattern 3: Fresh Session

Start a fresh session when the prior context is no longer reliable or has become too large and confusing.

Start fresh when:

- Tool results are stale.
- A large portion of the codebase changed.
- Requirements changed substantially.
- Extended context is degrading reasoning quality.
- The previous task and new task are fundamentally different.

A fresh session does not mean discarding all useful knowledge. Provide a concise prior-findings summary, but frame it as a hypothesis to validate.

Example:

```text
Prior analysis summary:
- The payment flow previously used module A.
- The earlier analysis identified a possible timeout issue.

The codebase has changed substantially since that analysis. Treat these
findings as hypotheses to validate, not established facts. Re-check the
current implementation before making recommendations.
```

### Critical Framing Rule

An injected summary in a fresh session is **a hypothesis to validate**, not an absolute fact. This prevents stale assumptions from being treated as current truth.

## Choosing the Correct Session Pattern

Use this decision guide:

```text
Are you continuing the same task with valid prior context?
    Yes -> Resume and describe specific changes.
    No  -> Continue.

Are you comparing independent approaches from a valid baseline?
    Yes -> Fork the session.
    No  -> Continue.

Are the prior results stale or is the context heavily invalidated?
    Yes -> Start fresh and inject prior findings as hypotheses.
```

| Situation | Correct pattern | Important action |
| --- | --- | --- |
| Same task, limited changes | Named resume | State what changed |
| Compare alternative approaches | Fork | Preserve the baseline |
| Stale results or major code changes | Fresh session | Validate injected hypotheses |

## Architectural Anti-Patterns

### 1. Passing Vague Prose Between Agents

Statements such as “use the previous findings” are not context passing. Include the actual structured findings in the task input.

### 2. Dropping Source Attribution

Do not summarize ten sources into one paragraph and pass it forward without source IDs, URLs, or claim mappings.

### 3. Silently Resolving Conflicts

Do not let a sub-agent discard one side of a disagreement. Preserve both claims and escalate the unresolved conflict.

### 4. Resuming Without Describing Changes

The resumed agent may reason from stale files, dependencies, or assumptions. State the specific changes that occurred.

### 5. Resuming a Heavily Changed Codebase

When most of the relevant system has changed, old context is more likely to mislead than help. Start fresh and inject prior findings as hypotheses.

### 6. Using Forks for Stale Context

Forking preserves a baseline; it does not refresh invalid information. Use a fresh session when the baseline itself is no longer trustworthy.

## Exam Scenarios

### Scenario 1: Generic Synthesis Output

**Question:** A synthesis sub-agent produces a generic report even though earlier agents found detailed evidence. What is the likely cause?

**Answer:** The coordinator passed a reference to the findings instead of the actual findings. Inject the structured data into the task prompt.

### Scenario 2: Lost Citations

**Question:** The final report contains accurate-looking summaries but no traceable sources. What should change?

**Answer:** Preserve claim-to-source mappings through every synthesis step and pass the complete claims-and-sources object to the next agent.

### Scenario 3: Conflicting Evidence

**Question:** Two sources disagree about the same topic. What should the sub-agent do?

**Answer:** Preserve both attributed claims, create an unresolved conflict object, and escalate it to the coordinator.

### Scenario 4: Recent Targeted Code Change

**Question:** A previous codebase analysis is still mostly valid, but the payments module was recently refactored. What should happen?

**Answer:** Resume the named session and explicitly identify the changed module so it can be re-analyzed.

### Scenario 5: Major Codebase Change

**Question:** The previous analysis is several weeks old and most modules have changed. Should the session be resumed?

**Answer:** Start a fresh session. Inject the old findings as hypotheses to validate, not as established facts.

### Scenario 6: Comparing Architectures

**Question:** You want to compare two implementations while preserving the original working analysis. Which pattern fits?

**Answer:** Fork the valid baseline session and explore each approach independently.

## Exam Checklist

- Remember that every sub-agent starts with a clean slate.
- Inject actual context into the task prompt; do not reference hidden coordinator history.
- Prefer structured context to vague prose summaries.
- Preserve claim IDs, source IDs, URLs, titles, dates, and confidence.
- Pass claims and sources together to every downstream agent.
- Never silently resolve conflicting source claims.
- Mark conflicts unresolved and escalate them to the coordinator.
- Resume a named session only when prior context remains valid.
- Tell a resumed session exactly what changed.
- Fork a valid baseline to compare independent approaches.
- Start fresh when results are stale or the codebase changed substantially.
- Frame injected prior findings in a fresh session as hypotheses to validate.
- Choose session management based on context validity, not on the amount of prior work invested.

## Final Summary

Sub-agent isolation is a design feature, but it places responsibility on the coordinator to pass context explicitly. Reliable systems use structured claim-and-source objects, preserve attribution, escalate conflicts, and synthesize results centrally.

For session management, **resume when the prior context is valid, fork when comparing approaches from a valid baseline, and start fresh when the old context is stale. Always describe changes during resumption and treat injected old findings as hypotheses in a fresh session.**