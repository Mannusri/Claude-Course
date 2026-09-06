# EP18: Why AI Agents Forget - Context Engineering

## Lesson Goal

This lesson explains why agents appear to forget information, how context engineering preserves critical facts, and when an agent should escalate a case to a human.

The main exam topics are:

- Context windows and token limits
- Stateless Claude API behavior
- Conversation history, tool calls, and tool results
- Lossy summarization
- Lost-in-the-middle attention behavior
- Case facts blocks
- Tool-result trimming with hooks
- Valid human-escalation triggers
- Escalation anti-patterns

## Why Agents Forget

An agent may forget an important fact not because the model's arithmetic failed, but because the relevant information was no longer accessible in the active context.

Example:

```text
Early conversation: Order 12345 costs $149.
Later conversation: The agent processes a refund.
Failure: The refund amount is wrong because the original amount was lost or buried.
```

This is a context-management failure.

## The Context Window

The context window is the model's active working memory for a request. It contains the information Claude can process during that turn.

It may include:

- System instructions
- User messages
- Assistant messages
- Tool-use blocks
- Tool-result blocks
- Conversation summaries
- Structured case facts
- Current task instructions

All of these consume tokens.

```text
System prompt + message history + tool calls + tool results = active context
```

A context window has a finite capacity. As the conversation and tool history grow, older or less accessible information can be dropped, compressed, or become harder for the model to use reliably.

## Claude API Statelessness

The Claude API is stateless. There is no automatic server-side memory that remembers previous requests for the application.

The application must send the relevant conversation history on every API call:

```text
Request 1: user message
Request 2: original user message + assistant response + tool result
Request 3: complete relevant history + new message
```

If the application omits an earlier message, Claude cannot access it in the next request.

### Exam Rule

When working with the raw Claude API, preserve and send the complete relevant history. Context engineering is an application responsibility.

## Why Tool Results Cause Context Growth

Agentic loops append tool calls and results on every iteration. Tool results can be verbose and may contain information that is useful only once.

Example sources of bloat:

- Full database records
- Repeated customer profiles
- Large search results
- Complete documents returned by a tool
- Debug logs
- Duplicate metadata
- Historical shipping details
- Repeated policy text

A few thousand unnecessary tokens per iteration can become a major context problem over many iterations.

## Lossy Summarization

Summarization can reduce context size, but careless summaries may remove facts required for correct decisions.

### Weak Summary

```text
Customer reported delivery and product issues and requested a refund.
```

This loses:

- Customer ID
- Order ID
- Product
- Amount
- Dates
- Policy window
- Exact failure details

### Fact-Preserving Summary

```text
Customer ID: C001
Order ID: 12345
Product: Wireless earphones
Amount: $149
Issue: Delivered three days late
Refund request: Full refund
Policy window: 30 days
Evidence: Delivery record confirms late arrival
```

### Exam Rule

A summary that loses identifiers, amounts, dates, or policy thresholds is a reliability failure, not merely a token-optimization issue.

## Lost in the Middle

Long contexts often have an attention pattern in which information at the beginning and end is easier for the model to use than information buried in the middle.

```text
High attention: beginning
Lower attention: middle
High attention: end
```

This does not mean the model always ignores the middle. It means critical information should not be placed only in a large block of low-priority historical content.

### Recommended Placement

```text
Beginning:
  Critical case facts and immutable identifiers

Middle:
  Trimmed tool results and lower-priority history

End:
  Current request, critical recap, policy reminder, and required action
```

Put the facts that control a decision where they are easy to retrieve.

## Case Facts Block

A case facts block is a compact, structured notepad containing verified transactional facts required for the current case.

Example:

```json
{
  "customer_id": "C001",
  "order_id": "12345",
  "item": "Wireless earphones",
  "refund_amount": 149,
  "order_date": "2026-08-20",
  "policy_window_days": 30,
  "customer_verified": true,
  "order_verified": true
}
```

### Properties of a Good Case Facts Block

- Contains only decision-relevant facts.
- Uses stable field names.
- Preserves exact IDs, dates, and amounts.
- Includes verification status.
- Is updated as new verified tool results arrive.
- Is placed in a high-attention position.
- Does not contain unsupported assumptions.

### Why It Helps

The case facts block reduces dependence on a long conversation summary. It provides a compact source of truth for the current decision.

## Tool-Result Trimming

Do not blindly append the entire raw result from every tool. Normalize and trim it before adding it to the next model request.

```text
Raw tool result
    -> PostToolUse normalization or extraction
    -> Verified case facts and relevant result
    -> Append compact context
```

A `PostToolUse` hook can:

- Remove irrelevant fields.
- Convert internal status codes.
- Extract IDs and amounts.
- Deduplicate repeated data.
- Normalize dates.
- Preserve source evidence.
- Update the case facts block.

A `PreToolUse` hook can validate or prepare the request before the tool executes.

### Important Timing Rule

Trim verbose tool output before it accumulates in the context. Cleaning the context only after it is already bloated is less effective.

## Evidence Board Pattern

Treat the case facts block as an evidence board:

```text
Verified facts -> Current case state -> Next decision
```

Only confirmed tool data should be promoted into the block. If a fact is uncertain, keep it marked as uncertain or route it for validation.

Do not let a model-generated guess become a permanent case fact without verification.

## Context Placement Pattern

A robust prompt layout can be:

```text
[Beginning]
Verified case facts
Non-negotiable policy constraints

[Middle]
Trimmed recent tool results
Relevant conversation excerpts

[End]
Current customer request
Critical recap
Required next action
```

This layout combines stable facts with current intent while keeping verbose history away from the most important attention zones.

## Context Compaction Versus Production Facts

A general conversation compaction can be useful for interactive coding sessions. However, production agents often need exact facts that a generic summary may omit.

Use structured state for:

- Customer IDs
- Order IDs
- Amounts
- Dates
- Policy thresholds
- Verification flags
- Authorization results
- Source references

Use summaries for narrative context that does not control a transactional decision.

## Escalation Patterns

Context quality affects escalation quality. An agent that has lost the customer, order, amount, or policy state cannot make a reliable autonomous decision.

Define explicit escalation triggers.

### Trigger 1: Explicit Human Request

If the customer asks to speak to a human, manager, or real person, escalate immediately.

Do not:

- Ask the customer to justify the request.
- Attempt another automated resolution first.
- Offer a self-service link instead.
- Continue collecting unnecessary details.

The explicit request is itself a valid escalation trigger.

### Trigger 2: Policy Gap or Exception

Escalate when the case is not clearly covered by the available policy or authorization rules.

Examples:

- Two policies conflict.
- The case combines conditions that no policy addresses together.
- The requested action exceeds the agent's authority.
- No approved decision path exists.
- A human judgment is required.

If the policy clearly covers the case and the agent can complete it, resolve it autonomously rather than escalating merely because it is complicated.

### Trigger 3: Agent Stuck After Reasonable Attempts

Escalate when the agent cannot make progress after bounded retries and reasonable recovery attempts.

Examples:

- Repeated tool failures.
- An unresolved validation loop.
- Missing required authorization.
- Contradictory verified data.
- A tool dependency remains unavailable.

The retry count should be bounded. Do not allow an endless loop to replace escalation.

## Invalid Escalation Triggers

### Negative Sentiment Alone

A customer may be angry, use capital letters, or include multiple exclamation marks while asking a question that the agent can resolve under clear policy.

Emotion alone is not a sufficient escalation trigger.

Escalate if the customer explicitly requests a human or if the task otherwise meets a valid trigger.

### Task Complexity Alone

A complex task does not automatically require a human if:

- The policy is clear.
- The tools are available.
- The agent is authorized.
- The agent is making progress.

Complexity can require more steps, not necessarily escalation.

### Model Confidence Score

Do not use the model's self-reported confidence as the primary escalation gate.

A model can be highly confident and wrong, or uncertain and correct. Confidence is not reliably calibrated to actual accuracy or policy authority.

Use explicit policies, validation, bounded retries, and human-request triggers instead.

## Escalation Decision Tree

```text
Did the customer explicitly request a human?
    Yes -> Escalate immediately.
    No  -> Continue.

Does an approved policy clearly cover the case?
    No  -> Escalate for human judgment.
    Yes -> Continue.

Is the agent authorized and able to progress?
    No  -> Escalate with a structured handoff.
    Yes -> Continue.

Have bounded recovery attempts been exhausted?
    Yes -> Escalate with evidence and retry history.
    No  -> Continue autonomously.
```

## Structured Handoff

An escalation should include a self-contained handoff summary:

```json
{
  "customer_id": "C001",
  "order_id": "12345",
  "verified_facts": {
    "refund_amount": 149,
    "policy_window_days": 30,
    "customer_verified": true
  },
  "reason": "customer_requested_human",
  "attempts": 0,
  "last_action": "customer explicitly requested a human agent",
  "next_action": "Connect to senior support"
}
```

The human should not need to reconstruct the case from a long transcript.

## Exam Scenarios

### Scenario 1: Lost Refund Amount

**Question:** An agent gives the wrong refund amount after a long conversation, even though the calculation logic is correct. What is the likely problem?

**Answer:** The required amount was lost or buried in the context. Preserve it in a verified case facts block and send it with the relevant request.

### Scenario 2: Verbose Tool Results

**Question:** Tool results consume thousands of tokens on every loop iteration. What should improve first?

**Answer:** Trim and normalize tool results with a post-tool hook before appending them to the next context, preserving only verified decision-relevant facts.

### Scenario 3: Critical Fact Placement

**Question:** Where should customer ID, order ID, refund amount, and policy threshold be placed in a long prompt?

**Answer:** In a compact high-attention case facts block near the beginning, with a critical recap near the end.

### Scenario 4: Explicit Human Request

**Question:** The customer says, “I do not want to deal with a bot. Connect me to a real person.” What should happen?

**Answer:** Escalate immediately without attempting another automated resolution.

### Scenario 5: Angry but Resolvable Customer

**Question:** The customer is angry but the request is clearly covered by policy and the agent is making progress. Should sentiment alone trigger escalation?

**Answer:** No. Negative sentiment alone is not a valid escalation trigger.

### Scenario 6: Policy Gap

**Question:** The case combines two conditions that are not covered by any approved policy. What should happen?

**Answer:** Escalate because a human must make a decision outside the defined policy.

### Scenario 7: Stuck Agent

**Question:** The agent has exhausted bounded retries and cannot resolve a repeated tool failure. What should happen?

**Answer:** Escalate with a structured handoff containing the verified facts, errors, attempts, and next action.

### Scenario 8: Model Confidence

**Question:** The model reports 95% confidence in a refund decision. Is that enough to avoid escalation?

**Answer:** No. Self-reported confidence is not a reliable authorization or escalation gate.

## Exam Anti-Patterns

### 1. Relying on Generic Summaries

A summary that omits IDs, dates, amounts, or thresholds can cause production decisions to use incomplete facts.

### 2. Appending Full Tool Results Forever

Verbose raw results create context bloat. Normalize and trim before adding them to the next iteration.

### 3. Putting Critical Facts Only in the Middle

Use a high-attention layout with facts at the beginning and a recap at the end.

### 4. Escalating Based Only on Sentiment

Anger does not prove that automation is impossible.

### 5. Escalating Based Only on Complexity

A complicated but clearly supported and authorized task can remain autonomous.

### 6. Using Confidence as Authorization

Model confidence is not a substitute for policy and validation.

### 7. Ignoring an Explicit Human Request

Do not force a customer through more automation after they clearly request a human.

### 8. Escalating Without Context

A human handoff that says only “please assist” forces the recipient to reconstruct the case. Include structured facts and reason.

## Exam Checklist

- The context window contains prompts, messages, tool calls, and tool results.
- The Claude API is stateless; send relevant history on every call.
- Preserve exact identifiers, numbers, dates, and policy thresholds.
- Use a compact verified case facts block as the source of truth.
- Put critical facts near the beginning and a recap near the end.
- Trim verbose tool results with hooks before they accumulate.
- Treat generic summaries as unsafe for transactional facts.
- Escalate on explicit human request immediately.
- Escalate when no approved policy covers the case.
- Escalate after bounded recovery attempts are exhausted.
- Do not escalate based only on sentiment, complexity, or model confidence.
- Return a structured, self-contained handoff summary.

## Final Summary

Agents do not forget because they have human memory; they lose access to facts when stateless requests, finite context windows, verbose tool results, lossy summaries, and attention placement are poorly managed. Use a compact verified case facts block, trim tool outputs before appending them, and place critical information at high-attention positions.

For escalation, **honor explicit human requests immediately, escalate policy gaps and exhausted failures, and do not use sentiment, complexity, or self-reported confidence as automatic triggers.**
