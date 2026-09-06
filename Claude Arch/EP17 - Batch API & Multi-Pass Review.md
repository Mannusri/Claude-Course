# EP17: Batch API and Multi-Pass Review

## Lesson Goal

This lesson explains when to use an asynchronous message-batch workflow and how to design independent multi-pass reviews that catch both local and cross-file problems.

The main exam topics are:

- Standard messages versus message batches
- Cost savings versus latency tradeoffs
- Non-blocking and blocking workflows
- The up-to-24-hour processing window
- `custom_id` request-response correlation
- Single-turn batch limitations
- Self-review bias
- Independent review sessions
- Per-file depth and cross-file breadth
- Prior findings and duplicate-review prevention

## Standard Messages Versus Batch Messages

### Standard Messages

The standard messages API is appropriate when the caller needs a response promptly and may need to continue interacting with Claude.

Use it when:

- A person is waiting for the result.
- A CI job needs the result for its next step.
- The workflow blocks a merge or deployment.
- The agent needs multiple tool calls.
- The agent needs multi-turn feedback.
- The result has a strict latency requirement.

```text
Request -> Immediate response -> Next step
```

### Message Batches

The message batches API accepts many independent requests for asynchronous processing. It is designed for workloads that can wait.

The tradeoff is lower cost in exchange for unpredictable completion timing within the supported processing window.

```text
Submit many requests
    -> Process asynchronously
    -> Retrieve results later
```

The transcript describes a discount of approximately 50% compared with standard processing. Verify current pricing and limits in the official Anthropic documentation because they can change.

## The Batch Decision Rule

Ask two questions:

1. Is anyone or any pipeline step waiting for the result now?
2. Does the task require multi-turn interaction or immediate tool feedback?

```text
Waiting now?                 -> Standard messages
Needs multi-turn interaction? -> Standard messages
No one waiting?              -> Continue
Independent single-turn work? -> Batch may fit
```

### Good Batch Use Cases

- Nightly code-review reports
- Overnight test generation
- Weekly customer-feedback classification
- Large independent document extraction
- Offline security audits
- Periodic reporting
- Processing thousands of independent records

### Poor Batch Use Cases

- Blocking pre-merge checks
- Immediate deployment approval
- A CI step whose output feeds the next step immediately
- Interactive customer support
- Agentic loops requiring tool results between turns
- Work requiring a person to respond to Claude

## Processing Window and No SLA

Batch processing is asynchronous. A request may complete quickly or may take much longer, up to the supported processing window.

For exam reasoning, treat the batch API as:

- No immediate response guarantee
- No useful latency SLA for blocking decisions
- A result that may arrive within up to 24 hours, according to the lesson scenario
- A workflow that must poll or retrieve results later

Never build a blocking workflow that assumes the batch result will arrive in seconds or minutes.

## Batch Request Correlation with `custom_id`

A batch can contain many requests. Each request needs a unique meaningful `custom_id` so the returned result can be associated with the original input.

Conceptually:

```json
{
  "custom_id": "pr-842-file-auth-service-v3",
  "params": {
    "model": "claude-model",
    "messages": [
      {
        "role": "user",
        "content": "Review this file for security issues."
      }
    ]
  }
}
```

When results are returned, the same identifier allows the application to correlate them:

```json
{
  "custom_id": "pr-842-file-auth-service-v3",
  "result": {
    "findings": []
  }
}
```

### Good IDs

Use identifiers that are unique and meaningful:

```text
pr-842-file-auth-service-v3
nightly-test-orders-2026-09-06
invoice-batch-2026-09-06-0042
```

Avoid relying only on opaque random IDs that make debugging and correlation harder. The ID still must be unique within the batch.

### Exam Rule

If a question asks how to match batch responses to their original requests, choose `custom_id`.

## Batch Interaction Limitations

Batch requests are suitable for independent work. They are not a replacement for a live agentic loop.

Do not use batches when the workflow requires:

- Immediate response handling
- Multiple back-and-forth turns
- Tool results fed into another model turn
- A user clarification between steps
- Streaming output
- A blocking gate with a short deadline

A batch request should be designed as an independent unit with all required input included at submission time.

## Batch Example: Nightly Analysis

A nightly job can classify 2,000 customer feedback forms:

```text
Cron trigger
    -> Create one batch request per form
    -> Assign unique custom_id values
    -> Submit the batch
    -> Retrieve results later
    -> Aggregate sentiment and topics
    -> Publish a report
```

This is a good fit because:

- Each form is independent.
- No user is waiting in real time.
- No multi-turn tool loop is required.
- The result can be reviewed the next day.
- Cost savings matter more than immediate latency.

## Batch Example: Incorrect Pre-Merge Design

A pull request cannot merge until Claude approves it:

```text
PR opened
    -> Submit batch review
    -> Wait for batch result
    -> Decide whether to merge
```

This is a poor fit because the merge is blocked and the batch may not complete within the CI job's timeout or required latency.

Use the standard messages API for the blocking review.

## Self-Review Bias

A Claude instance that writes code and then reviews its own code in the same session retains the reasoning behind its implementation.

It may:

- Rationalize earlier choices
- Assume its design is correct
- Focus on expected happy paths
- Miss flaws that an independent reviewer would notice
- Repeat the same assumptions in generated tests

```text
Instance A writes code
Instance A reviews code -> Biased by its own context
```

A fresh independent session provides a different perspective:

```text
Instance A writes code
Instance B reviews code -> Fresh context and independent reasoning
```

### Exam Rule

Do not ask the same session to review its own work more carefully as the primary quality solution. Use an independent review instance.

## Independent Review Sessions

A fresh review session should receive the current files and the project-level rules it needs, but not the full implementation reasoning that could bias it.

Useful context includes:

- Current repository state
- Changed files
- Project `CLAUDE.md`
- Review criteria
- Prior findings for deduplication
- Required output schema

Avoid passing the entire implementation conversation when the goal is independent critique.

## Multi-Pass Review Architecture

A single broad review of an entire pull request can be shallow. A multi-pass architecture combines focused local review with system-level integration review.

```text
Changed files
    |
    +--> Independent per-file reviews in parallel
    |       +--> Style and correctness
    |       +--> Security
    |       +--> Tests
    |
    v
Collated findings
    |
    v
Cross-file integration review
    |
    v
Final synthesized report
```

## Per-File Review: Depth

Independent per-file reviewers focus deeply on a bounded unit.

They can inspect:

- Local logic
- Error handling
- Security checks
- Tests for the file
- Input validation
- Side effects
- Style or project-specific criteria

Because each instance has a smaller scope, it can spend more reasoning on the file rather than scanning a huge pull request.

Independent reviews can run in parallel when the files do not depend on one another for the first pass.

## Cross-File Review: Breadth

The integration pass receives the collated local findings and relevant file relationships.

It checks:

- Imports and exports
- API contracts
- Shared types
- Authentication flow across modules
- Data ownership
- State transitions
- Dependency compatibility
- Tests that span multiple components
- Whether a change in one file breaks another

A file can be locally correct while the feature is globally broken. The cross-file pass catches those integration failures.

### Depth and Breadth

```text
Per-file passes -> Depth within individual files
Integration pass -> Breadth across the feature
```

The two stages complement each other.

## Multi-Pass Review with Prior Findings

CI often creates a fresh session on every pull-request update. Without prior findings, the new session may repeat comments that were already reported.

Use this workflow:

1. Fetch previous review comments.
2. Associate them with the pull request and file.
3. Pass them to the new per-file or integration review.
4. Ask Claude to report only new or materially changed findings.
5. Store the new findings with stable identifiers.

Example instruction:

```text
These findings were reported in an earlier review. Do not repeat unchanged
findings. Reassess them only if the new commit changes the evidence, severity,
location, or remediation. Report only new or materially changed findings.
```

## Enterprise Review Architecture

### Blocking Pull-Request Pipeline

Use standard messages:

```text
PR opened or updated
    -> Load prior findings
    -> Independent per-file reviews
    -> Cross-file integration review
    -> Validate structured output
    -> Block or allow merge
```

This workflow needs timely results and may require multiple stages, so standard synchronous processing is the appropriate choice.

### Nightly or Offline Pipeline

Use message batches:

```text
Nightly schedule
    -> Submit independent requests with custom IDs
    -> Retrieve results later
    -> Aggregate reports
    -> Notify the team next morning
```

This workflow is non-blocking and can trade latency for cost.

## Choosing Standard Versus Batch

| Requirement | Standard messages | Message batches |
| --- | --- | --- |
| Human waiting now | Best choice | Poor fit |
| Blocking merge gate | Best choice | Poor fit |
| Multi-turn tool loop | Best choice | Poor fit |
| Immediate downstream step | Best choice | Poor fit |
| Nightly independent jobs | Possible but expensive | Best choice |
| Large offline workload | Possible but expensive | Best choice |
| Streaming | Suitable where supported | Not the intended model |
| Cost optimization with flexible timing | Less favorable | Strong fit |
| Request-response correlation | Message IDs | `custom_id` per request |

## Exam Scenarios

### Scenario 1: Nightly Test Generation

**Question:** A nightly job generates tests for thousands of independent changed files. No user is waiting, and results are reviewed the next day. Which API fits?

**Answer:** Message batches, with a unique `custom_id` for each request.

### Scenario 2: Blocking Merge Review

**Question:** A pull request cannot merge until Claude returns security findings. Should the team switch to batches to save cost?

**Answer:** No. The workflow is blocking and needs timely results. Use standard messages.

### Scenario 3: Multi-Turn Agent

**Question:** An agent must call a tool, inspect its result, call another tool, and use feedback across turns. Which API fits?

**Answer:** Standard messages. A batch is not the right mechanism for an interactive multi-turn loop.

### Scenario 4: Correlating Results

**Question:** A batch contains hundreds of invoice requests. How does the application match each result to the source invoice?

**Answer:** Assign a unique meaningful `custom_id` to every request and use it when processing the result.

### Scenario 5: Self-Review

**Question:** Claude writes an authentication module and then reviews it in the same session. What is the primary risk?

**Answer:** The review is biased by the session's original reasoning. Use an independent fresh review session.

### Scenario 6: Shallow PR Review

**Question:** One model reviews an entire large pull request and misses cross-file problems. What architecture improves coverage?

**Answer:** Run independent per-file passes for depth, collate the findings, then run a cross-file integration pass for breadth.

### Scenario 7: Duplicate Comments

**Question:** Every new commit causes the reviewer to repeat all old findings. What should be added?

**Answer:** Fetch and pass prior findings into the fresh review session, asking it to report only new or materially changed findings.

## Exam Anti-Patterns

### 1. Batch for a Blocking Gate

Do not use an asynchronous up-to-24-hour workflow when a merge or deployment is waiting.

### 2. Batch for Multi-Turn Tool Use

A live agentic loop needs immediate feedback and subsequent turns. Use standard messages.

### 3. Omitting `custom_id`

Without a stable per-request identifier, correlating batch responses becomes unreliable.

### 4. Random Opaque Correlation Only

Use unique, meaningful IDs that make logs and result mapping understandable.

### 5. Same-Session Self-Review

A model reviewing its own implementation in the same context is biased. Use independent instances.

### 6. Only Per-File Review

Per-file analysis can miss integration failures. Add a cross-file synthesis pass.

### 7. Only Whole-PR Review

A single broad pass can be shallow. Use focused parallel passes before integration review.

### 8. Repeating Prior Findings

Fresh sessions need prior review context to avoid duplicate comments and wasted tokens.

## Exam Checklist

- Use message batches for asynchronous, independent, non-blocking workloads.
- Expect flexible completion timing and the lesson's up-to-24-hour window.
- Treat batch processing as a cost-versus-latency tradeoff.
- Use standard messages when a person or pipeline is waiting.
- Use standard messages for multi-turn tool interactions.
- Assign a unique meaningful `custom_id` to every batch request.
- Use `custom_id` to correlate each response with its request.
- Do not assume batch results are available immediately.
- Use fresh independent sessions for code generation and review.
- Use per-file passes for depth.
- Use a cross-file integration pass for breadth.
- Pass prior findings to fresh CI sessions to prevent duplicate comments.
- Keep blocking review gates on timely processing paths.

## Final Summary

Message batches reduce cost for flexible, asynchronous, independent work, but they are not suitable for blocking pipelines, immediate downstream decisions, or multi-turn agentic loops. Each request needs a unique `custom_id` for response correlation.

For review quality, use independent fresh sessions: run parallel per-file reviews for depth, then a cross-file integration pass for breadth. Pass prior findings to prevent repeated comments.

For the exam, remember: **batch for non-blocking work, standard messages for waiting or multi-turn work, `custom_id` for correlation, and independent multi-pass review for unbiased depth and breadth.**
