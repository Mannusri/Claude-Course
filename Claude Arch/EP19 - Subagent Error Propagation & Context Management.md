# EP19: Subagent Error Propagation and Context Management

## Lesson Goal

This lesson explains how sub-agents should report failures to coordinators and how to preserve useful context across long or interrupted Claude Code sessions.

The main exam topics are:

- Tool errors versus sub-agent errors
- Structured sub-agent-to-coordinator propagation
- Error category and retryability
- Attempted query, partial results, and alternatives
- Access failure versus valid empty results
- Silent error suppression
- Immediate pipeline termination
- Local recovery before escalation
- Scratchpad files
- `/compact`
- Manifest checkpoints

## Two Error Boundaries

There are two related but different error-propagation layers.

### Tool to Sub-Agent

A tool reports failure to the agent that called it. This layer uses concepts such as:

- `is_error`
- Error category
- Retryability
- Tool-level description
- Tool result metadata

### Sub-Agent to Coordinator

A sub-agent may perform local retries and use several tools. If it still cannot complete its task, it must report a structured failure to the coordinator.

```text
Tool failure
    -> Sub-agent classifies and recovers locally
    -> Sub-agent returns success or structured failure
    -> Coordinator chooses retry, fallback, reassignment, or escalation
```

The coordinator should not need to reconstruct the sub-agent's hidden history to understand what happened.

## Why Generic Errors Fail

A result such as this is not actionable:

```json
{
  "is_error": true,
  "message": "Operation failed"
}
```

The coordinator cannot determine:

- What kind of failure occurred
- Whether retrying is reasonable
- What query was being processed
- What work was already completed
- Which alternatives exist
- Whether the failure is an access problem or a valid empty result

Generic errors can cause the coordinator to silently abandon work, retry blindly, or terminate an otherwise recoverable pipeline.

## Recovery-Ready Error Contract

A sub-agent error should include at least:

1. `is_error`
2. `error_category`
3. `is_retryable`
4. `attempted_query`
5. `partial_results`
6. `alternatives`
7. A clear description or next action

Example:

```json
{
  "is_error": true,
  "error_category": "timeout",
  "is_retryable": true,
  "attempted_query": "Find five recent revenue studies from the web corpus.",
  "partial_results": [
    "document-a",
    "document-b",
    "document-c"
  ],
  "alternatives": [
    "company-sec-filings",
    "internal-research-database"
  ],
  "description": "Web search timed out after 30 seconds while retrieving the remaining documents.",
  "next_action": "Retry the missing portion, then use an alternative source if the retry fails."
}
```

## Meaning of Each Field

### `is_error`

Clearly tells the coordinator that the sub-agent did not complete normally.

### `error_category`

Explains the failure type, such as:

- `timeout`
- `permission`
- `validation`
- `business_logic`
- `no_results`
- `internal`
- `service_unavailable`

### `is_retryable`

Indicates whether another attempt could reasonably succeed.

Examples:

- Timeout: often retryable
- Temporary service outage: often retryable
- Invalid query: retry only after correction
- Permission denied: not retryable without changed authorization
- Valid empty result: not retryable with the same query and source

### `attempted_query`

Describes what the sub-agent was trying to do when it failed. This allows the coordinator to understand the missing work.

### `partial_results`

Preserves useful work completed before failure. A failed sub-agent may still have retrieved or validated valuable data.

### `alternatives`

Lists fallback sources, tools, agents, or recovery paths the coordinator can use.

### Description and Next Action

Provides human- and model-readable context without requiring the coordinator to inspect an entire sub-agent transcript.

## Partial Results Must Survive Failure

A sub-agent that retrieves three of five required documents has not completed the task, but its three documents remain valuable.

The coordinator can:

1. Keep the three valid documents.
2. Retry only the missing portion.
3. Use an alternative source for the remaining documents.
4. Assign a second sub-agent to complete the gap.

Do not discard all work merely because the final task was incomplete.

## Access Failure Versus Valid Empty Result

These outcomes look similar but require different actions.

### Access Failure

The source could not be checked successfully.

Examples:

- Database unavailable
- Network timeout
- Authentication service failed
- Permission denied
- Remote server unavailable

The query did not complete reliably. Depending on the category, retry or use a fallback.

```json
{
  "is_error": true,
  "error_category": "timeout",
  "is_retryable": true,
  "partial_results": []
}
```

### Valid Empty Result

The source was accessed successfully, the query ran, and nothing matched.

```json
{
  "is_error": false,
  "error_category": "no_results",
  "is_retryable": false,
  "results": []
}
```

The coordinator should not waste retries on a query that successfully found no matches.

### Comparison

| Outcome | Was the source accessed? | Retry same request? | Meaning |
| --- | --- | --- | --- |
| Access failure | No or unreliably | Often yes | The query may still succeed later |
| Valid empty result | Yes | Usually no | The query completed and found nothing |

### Exam Rule

Do not represent an access failure as a successful empty result. The error category and retryability must distinguish them.

## Local Recovery Before Propagation

A sub-agent should attempt reasonable local recovery before escalating to the coordinator.

```text
Transient tool failure
    -> Local retry with bounded attempts
    -> Adjust query or format if appropriate
    -> Try an approved fallback source
    -> Structured escalation to coordinator
```

Examples:

- Retry a timeout once or twice.
- Correct malformed input before retrying.
- Try a backup source if the primary is unavailable.
- Preserve partial results before propagating the failure.

The coordinator should become involved when local recovery options are exhausted or when a higher-level decision is required.

## Coordinator Responsibilities

The coordinator decides what to do with a structured sub-agent failure:

- Retry the sub-agent.
- Retry only the missing portion.
- Invoke an alternative tool.
- Delegate to another sub-agent.
- Continue using partial results.
- Escalate to a human.
- Terminate the workflow when continuation is unsafe.

A sub-agent should not unilaterally terminate the entire multi-agent pipeline unless the architecture explicitly grants that authority.

## Error Anti-Pattern 1: Silent Suppression

Silent suppression occurs when a sub-agent encounters an exception but returns a success-looking response such as:

```json
{
  "status": "ok",
  "results": []
}
```

The coordinator interprets this as a valid empty result and continues confidently with incomplete data.

Consequences:

- The failure is hidden.
- Partial work is lost.
- Retry decisions are impossible.
- Reports may be confidently wrong.
- Observability is destroyed.

Always propagate the failure explicitly.

## Error Anti-Pattern 2: Immediate Termination

Immediate termination occurs when one sub-agent failure aborts the entire pipeline before the coordinator can evaluate recovery options.

Consequences:

- Other successful sub-agents' work is discarded.
- Partial results are lost.
- Fallbacks are never attempted.
- The coordinator cannot make an informed decision.

Use structured propagation instead. The coordinator should own the final decision to continue or terminate.

## Context Degradation in Claude Code

In a long Claude Code session, every file read, tool call, shell command, and result contributes to the conversation history.

Over time:

- Early findings may fall outside the active context.
- Cross-file relationships may be forgotten.
- Claude may reread files unnecessarily.
- Tokens are spent rediscovering known information.
- The agent's reasoning becomes less reliable.

This is context degradation, not necessarily a code defect.

## Scratchpad Files

A scratchpad is a persistent Markdown or JSON file where Claude records important findings during exploration.

Example location:

```text
.claude/scratchpad.md
```

The scratchpad can contain:

- Files inspected
- Key architectural findings
- Important dependencies
- Confirmed assumptions
- Open questions
- Decisions already made
- Remaining work

### How It Helps

When the conversation context becomes crowded, Claude can reread the compact scratchpad instead of rereading the entire codebase.

```text
Explore codebase
    -> Write verified findings to scratchpad
    -> Context fills
    -> Reread scratchpad
    -> Continue with recovered context
```

### Activation Requirement

Claude Code does not automatically create a useful scratchpad merely because one exists as a concept. Instruct it to maintain one.

Interactive prompt example:

```text
As you explore this project, continuously record verified findings,
files inspected, open questions, and decisions in .claude/scratchpad.md.
Keep the file concise and update it after each major discovery.
```

For automated work, put the instruction in project-level `CLAUDE.md` or implement the behavior in the agent logic.

## Scratchpad Guidelines

A useful scratchpad should be:

- Concise
- Structured
- Fact-based
- Updated continuously
- Stored at a predictable path
- Safe to reread
- Free of unsupported guesses

Do not let it become a second unstructured transcript. It should preserve the facts needed to continue work.

## `/compact`

The `/compact` command compresses an interactive Claude Code conversation into a summary to make room for continued work.

Use it when:

- The interactive session is approaching its context limit.
- The current conversation contains useful history but too much raw detail.
- You need to continue the same general task with a smaller context.

### Limitation

A generic compact summary may omit critical IDs, exact values, or subtle dependencies. Preserve those in a scratchpad or structured state when exact recovery matters.

The command is primarily an interactive Claude Code mechanism. Do not assume it can be issued as a normal runtime parameter in an unattended CI job. For CI, use explicit files, summaries, manifests, and programmatic context management.

## Manifest Files

A manifest is a persistent checkpoint describing the current state of a long-running exploration or implementation task.

Example location:

```text
.claude/manifest.json
```

Example:

```json
{
  "task": "Map the billing service and prepare a migration plan",
  "status": "exploration_in_progress",
  "completed_steps": [
    "Mapped packages/billing",
    "Reviewed payment interfaces",
    "Traced invoice repository"
  ],
  "current_step": "Review retry policy implementation",
  "next_steps": [
    "Inspect payment-worker.ts",
    "Compare integration tests",
    "Update migration plan"
  ],
  "last_updated": "2026-09-06T23:00:00Z"
}
```

### Scratchpad Versus Manifest

| Artifact | Purpose |
| --- | --- |
| Scratchpad | Ongoing findings, facts, and open questions |
| Manifest | Checkpoint of completed work, current state, and next steps |

A scratchpad answers “What have we learned?” A manifest answers “Where exactly are we in the workflow?”

### Recovery Workflow

```text
Before a major step -> Update manifest
Session crashes     -> Start a new session
New session          -> Read manifest and scratchpad
Continue             -> Resume from the recorded checkpoint
```

Use a predictable location under `.claude/` so the next session can find the files without searching the entire machine.

## Reliability Layers

A robust architecture can combine three layers:

### Error Layer

Structured propagation from sub-agent to coordinator:

- Error category
- Retryability
- Attempted query
- Partial results
- Alternatives
- Description and next action

### Context Layer

Persistent context management:

- Scratchpad findings
- Compact summaries for interactive sessions
- Trimmed and structured tool results

### Recovery Layer

Restart and continuation support:

- Manifest checkpoints
- Predictable file locations
- Recorded completed and next steps

```text
Structured error -> Persistent findings -> Recoverable checkpoint
```

## Exam Scenarios

### Scenario 1: Generic Sub-Agent Failure

**Question:** A research sub-agent returns “operation failed,” and the coordinator cannot decide whether to retry or use another source. What should change?

**Answer:** Return structured error context containing category, retryability, attempted query, partial results, alternatives, and a clear description.

### Scenario 2: Partial Research

**Question:** A sub-agent retrieved three of five documents before a timeout. What should the coordinator receive?

**Answer:** The timeout metadata, retryability, the three partial documents, the attempted query, and possible alternative sources.

### Scenario 3: Empty Search

**Question:** The search service completed normally but found no matching documents. Should the coordinator retry automatically?

**Answer:** Treat it as a valid empty result with `is_retryable: false`, unless a changed query or new information justifies a new search.

### Scenario 4: Database Unavailable

**Question:** The database was unreachable during a search. How should it be represented?

**Answer:** As an access or timeout failure with an explicit error flag and retryable metadata when retrying or using a fallback may succeed.

### Scenario 5: Silent Suppression

**Question:** A sub-agent catches an exception and returns `{status: "ok", results: []}`. What is wrong?

**Answer:** The error was silently suppressed and converted into a success-looking empty result. Propagate the structured failure.

### Scenario 6: Pipeline Crash

**Question:** One sub-agent fails and immediately terminates all other work. What is the anti-pattern?

**Answer:** Immediate termination. Propagate the error to the coordinator so it can preserve partial results and choose the recovery path.

### Scenario 7: Long Claude Code Session

**Question:** Claude repeatedly rereads files after forgetting earlier findings. What should be added?

**Answer:** Instruct Claude to maintain a concise scratchpad file with verified findings and open questions.

### Scenario 8: Interrupted Exploration

**Question:** A terminal closes halfway through a multi-hour codebase exploration. How can the next session resume from a known point?

**Answer:** Read a predictable `.claude/manifest.json` checkpoint together with the scratchpad.

### Scenario 9: CI Context Management

**Question:** An unattended CI session needs persistent exploration state. Should it rely on typing `/compact` interactively?

**Answer:** No. Use committed or generated scratchpad and manifest files, or implement context management programmatically.

## Exam Checklist

- Distinguish tool-to-agent errors from sub-agent-to-coordinator errors.
- Never return generic “operation failed” context.
- Include error category, retryability, attempted query, partial results, and alternatives.
- Preserve useful partial results after failure.
- Distinguish access failure from a successful empty result.
- Do not silently suppress exceptions as successful empty output.
- Do not let one sub-agent immediately terminate the whole pipeline.
- Try bounded local recovery before escalating to the coordinator.
- Use scratchpad files for persistent verified findings during exploration.
- Instruct Claude Code to maintain the scratchpad; it is not automatic by default.
- Use `/compact` for interactive context compression, with care around lost facts.
- Use a manifest for checkpoints, completed steps, and restart recovery.
- Store scratchpads and manifests at predictable `.claude/` paths.
- Use structured files and programmatic context management in CI.

## Final Summary

Reliable multi-agent systems propagate structured failures rather than vague status messages. A sub-agent should classify the error, state retryability, describe its attempted query, preserve partial results, and offer alternatives so the coordinator can decide what to do.

For long Claude Code work, use a scratchpad for verified findings, `/compact` for interactive compression, and a manifest for recoverable checkpoints. **Never turn an access failure into an empty success, never discard partial work, and never let a worker terminate the whole pipeline without coordinator judgment.**
