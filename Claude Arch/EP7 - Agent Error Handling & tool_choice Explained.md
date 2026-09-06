# EP07: Agent Error Handling and `tool_choice` Explained

## Lesson Goal

This lesson explains how to make agentic systems recover from tool failures, communicate actionable error state, limit tool access, and control whether Claude must use a tool.

The main exam topics are:

- Explicit tool-error signaling
- `is_error` and structured error metadata
- Transient, validation, business-logic, and permission failures
- Retryable versus non-retryable errors
- Local recovery before coordinator escalation
- Scoped tool access and least privilege
- The `tool_choice` modes `auto`, `any`, and `tool`
- Common error-handling anti-patterns

## Why Tool Errors Are Dangerous

When a tool fails, the model must be told explicitly that the result is an error. Otherwise, the model may interpret an empty result or an error-shaped payload as a successful result and confidently continue with incorrect reasoning.

```text
Tool failure
    -> Explicit error result
    -> Model understands the failure
    -> Model chooses a recovery path
```

Do not silently return an empty array, empty object, or normal-looking success message when the tool failed. The model needs a machine-readable failure signal and enough context to decide whether to retry, use an alternative, or escalate.

## The `is_error` Flag

A failed tool result should set:

```json
{
  "is_error": true
}
```

The result should also contain structured metadata describing the failure. `is_error: true` tells the model that the operation failed; the metadata tells it what to do next.

Conceptual result:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_123",
  "is_error": true,
  "content": {
    "error_category": "transient",
    "is_retryable": true,
    "description": "The order service timed out.",
    "retry_after_ms": 2000,
    "alternatives": ["lookup_order_cache"]
  }
}
```

The exact field names depend on the application's contract. The exam principle is that the error must be explicit, categorized, and actionable.

## Structured Error Metadata

A useful error response should answer:

- What failed?
- What category of failure occurred?
- Is retrying appropriate?
- Should the agent use an alternative tool?
- Should the coordinator or a human be involved?
- What context is useful for recovery or debugging?

Recommended metadata:

| Field | Purpose |
| --- | --- |
| `is_error` | Signals that the tool operation failed |
| `error_category` | Classifies the failure and guides recovery |
| `is_retryable` | Indicates whether retrying can reasonably help |
| `description` | Human- and model-readable explanation |
| `retry_after_ms` | Optional delay for a retryable transient failure |
| `alternatives` | Optional fallback tools or recovery paths |
| `details` | Safe diagnostic context |

### Generic Error: Weak

```json
{
  "is_error": true,
  "content": "Operation failed."
}
```

The coordinator cannot determine whether to retry, fix input, escalate, or choose another tool.

### Actionable Error: Strong

```json
{
  "is_error": true,
  "content": {
    "error_category": "permission",
    "is_retryable": false,
    "description": "The agent is not authorized to access the billing database.",
    "next_action": "Escalate to an authorized billing agent."
  }
}
```

## Four Important Tool-Failure Categories

### 1. Transient Failure

A temporary condition may succeed later.

Examples:

- Network timeout
- Temporary database outage
- Rate limit
- Service unavailable
- Intermittent connection failure

Typical metadata:

```json
{
  "error_category": "transient",
  "is_retryable": true,
  "retry_after_ms": 2000
}
```

Retrying is appropriate, preferably with a bounded attempt count and a delay or backoff.

### 2. Validation Failure

The tool received invalid, malformed, or incomplete input.

Examples:

- Invalid customer ID format
- Missing required order ID
- Unsupported date format
- Amount outside the accepted input schema

Retrying the exact same request is not useful. A retry may be appropriate only after correcting the input.

Exam nuance:

- Same invalid input again: not useful
- Corrected input: retry may succeed
- The error should explain what must be fixed

### 3. Business-Logic or Policy Failure

The request is understood but not allowed by business rules.

Examples:

- Refund exceeds the agent authorization limit
- Order is not eligible for a refund
- Customer is not verified
- Operation violates a compliance rule

Typical metadata:

```json
{
  "error_category": "business_logic",
  "is_retryable": false,
  "description": "Refund exceeds the automatic authorization limit.",
  "next_action": "Escalate to senior support."
}
```

Do not retry the same prohibited action. Route it to the correct approval or escalation path.

### 4. Permission Failure

The agent or runtime is not authorized to access the tool or resource.

Examples:

- Missing permission to call a billing tool
- Agent cannot access a database
- Resource belongs to another tenant
- Tool is not in the agent's allowed-tool set

Typical metadata:

```json
{
  "error_category": "permission",
  "is_retryable": false,
  "description": "Access denied for lookup_billing_account."
}
```

Retrying without changing authorization does not solve the problem. Use an authorized agent, request approval, or escalate.

## Retry Decision Matrix

| Failure category | Retry the same request? | Correct recovery |
| --- | --- | --- |
| Transient | Yes, with a bounded retry | Delay, retry, or use a fallback |
| Validation | No, unless input is corrected | Fix parameters and retry |
| Business logic | No | Change the business state or escalate |
| Permission | No | Use authorized access or escalate |
| Unknown/unexpected | Usually no automatic retry | Capture context and escalate safely |

The exact retry policy may vary, but the exam distinction is consistent: transient failures are normally retryable; policy and permission failures are not; validation failures require corrected input.

## Local Recovery Before Escalation

A sub-agent should try reasonable local recovery before propagating a recoverable error to the coordinator.

```text
Tool fails with transient error
        |
        v
Sub-agent performs one or two bounded local retries
        |
   succeeds? ---- yes ----> Continue the sub-agent loop
        |
        no
        v
Propagate structured error to coordinator
        |
        v
Coordinator chooses fallback, reassignment, or escalation
```

### Why Local Recovery Helps

- The sub-agent understands the immediate tool context.
- Minor transient failures do not interrupt the whole workflow.
- The coordinator is reserved for decisions that exceed the worker's scope.
- The system avoids unnecessary escalation for recoverable problems.

Retries must be bounded. An unbounded retry loop creates a different failure mode.

### Pipeline Anti-Pattern

Do not terminate the entire pipeline on the first untried transient failure. A single failure should not automatically destroy a multi-agent workflow when local recovery is possible.

## Centralized Tool-Call Handling

A reliable implementation can centralize the sequence that surrounds a tool call:

```text
Apply PreToolUse hooks
    -> If blocked, construct structured policy error
    -> Otherwise dispatch the tool
    -> Catch and classify exceptions
    -> Apply PostToolUse hooks on successful results
    -> Return success or structured error result
```

This keeps policy checks, tool dispatch, result normalization, and error construction consistent across the agentic loop.

Conceptual pseudocode:

```python
def handle_tool_call(tool_name, tool_input, session_state):
    gate = apply_pre_tool_hooks(tool_name, tool_input, session_state)
    if not gate["allowed"]:
        return make_error_result(
            category=gate["category"],
            retryable=False,
            description=gate["reason"],
        )

    try:
        raw_result = dispatch_tool(tool_name, tool_input)
        return apply_post_tool_hooks(tool_name, raw_result, session_state)
    except TimeoutError as error:
        return make_error_result(
            category="transient",
            retryable=True,
            description=str(error),
        )
    except PermissionError as error:
        return make_error_result(
            category="permission",
            retryable=False,
            description=str(error),
        )
```

The important design principle is consistent structured signaling, not the exact programming language or exception class names.

## Scoped Tool Access

Giving an agent more tools does not automatically make it more capable. A large, overlapping tool set increases the number of choices Claude must distinguish during inference and can reduce selection reliability.

### Unscoped Design

```text
Coordinator: 20 unrelated tools
```

This can cause:

- Wrong tools firing
- More ambiguous routing
- Hard-to-debug behavior
- Unnecessary permission exposure
- Larger tool-selection context

### Scoped Design

```text
Coordinator agent: get_customer, escalate_to_human
Order agent: lookup_order, process_refund
Communication agent: send_email, create_ticket
Research agent: search_live_web, fetch_url, extract_content
Synthesis agent: verify_fact
```

Each agent receives only tools relevant to its role.

### Least Privilege

Scoped tool access supports least privilege:

- The agent sees fewer irrelevant choices.
- Unauthorized operations are less likely.
- Tool routing becomes more reliable.
- The architecture is easier to audit.

For a synthesis agent that only needs to verify a claim, provide a purpose-built `verify_fact` tool rather than full web-search access.

## The `tool_choice` Parameter

Tool scoping limits what an agent can use. `tool_choice` controls how Claude chooses among the tools that are available in the request.

The key modes are:

- `auto`
- `any`
- `tool`

## `tool_choice: auto`

```json
{
  "type": "auto"
}
```

The model decides whether to call a tool and, if so, which available tool to call.

Use `auto` when:

- A text response may be appropriate.
- Tool use is optional.
- The model should select among several tools based on the request.

Important: `auto` does not guarantee that a tool will be called.

## `tool_choice: any`

```json
{
  "type": "any"
}
```

The model must call at least one available tool, but it chooses which tool.

Use `any` when:

- Tool use is mandatory.
- You need structured output through a tool.
- A text-only response is not allowed.
- Several tools are acceptable, but the application does not want to select one exact tool.

`any` means “choose one of the available tools,” not “choose this named tool.”

## `tool_choice: tool`

```json
{
  "type": "tool",
  "name": "lookup_order"
}
```

The model must call the exact named tool.

Use `tool` when:

- One specific operation must happen.
- A particular structured schema is required.
- The application already knows which tool is appropriate.
- The model should fill the selected tool's arguments rather than choose the tool.

## Tool-Choice Comparison

| Mode | Who chooses the tool? | Is a tool call guaranteed? | Typical use |
| --- | --- | --- | --- |
| `auto` | Model, if it chooses to call one | No | Optional tool use and normal conversation |
| `any` | Model chooses among available tools | Yes, at least one | Mandatory structured tool output |
| `tool` | Application names the tool | Yes, that exact tool | Enforced operation or schema |

### Common Exam Signal

If the question says “the model must produce structured output using one of the available tools,” choose `any`.

If it says “the model must call this exact tool,” choose `tool` and provide the tool name.

If it says “let the model decide whether and which tool to use,” choose `auto`.

## Error Handling and `tool_choice` Together

These mechanisms solve different problems:

```text
Scoped tools       -> Which tools are visible?
tool_choice        -> Whether and which tool must be called?
is_error metadata  -> What happened when the tool was called?
Local recovery     -> What can the current agent retry?
Coordinator        -> What should happen after local recovery fails?
```

Do not use `tool_choice` as a replacement for error metadata, and do not use error metadata as a replacement for scoped permissions.

## Exam Scenarios

### Scenario 1: Database Timeout

**Question:** An order lookup times out temporarily. What should the tool result contain?

**Answer:** Set `is_error: true`, classify the error as `transient`, mark it retryable, and include useful retry or fallback metadata.

### Scenario 2: Refund Policy Violation

**Question:** A refund exceeds the automatic authorization limit. Should the agent retry the same call?

**Answer:** No. Return a structured business-logic error and route the case to the appropriate approval or human-escalation path.

### Scenario 3: Invalid Input

**Question:** An order ID is malformed. Is retrying allowed?

**Answer:** Retrying the same value is not useful. Correct the input first; then a new attempt may be valid.

### Scenario 4: Access Denied

**Question:** A worker does not have access to the billing tool. Should it retry?

**Answer:** No. Use an authorized agent or escalate. Retrying without changing permission will not solve the failure.

### Scenario 5: Local Recovery

**Question:** A sub-agent receives a transient timeout. Should it immediately escalate to the coordinator?

**Answer:** It should perform a bounded local retry first. Escalate a structured error only after local recovery is exhausted.

### Scenario 6: Mandatory Tool Use

**Question:** The model must call one of several tools to return structured output, but the application does not care which one. Which `tool_choice` mode fits?

**Answer:** `any`.

### Scenario 7: Exact Tool Use

**Question:** The application requires Claude to call `extract_order_data` specifically. Which mode fits?

**Answer:** `tool` with the exact tool name.

### Scenario 8: Optional Tool Use

**Question:** Claude may answer directly or choose a relevant tool when needed. Which mode fits?

**Answer:** `auto`.

### Scenario 9: Synthesis Agent Reliability

**Question:** A synthesis agent is confused because it has access to twenty unrelated tools. What is the best improvement?

**Answer:** Scope its allowed tools to a small, role-specific set, such as a purpose-built `verify_fact` tool.

## Exam Anti-Patterns

### 1. Empty Success-Looking Results

Do not return an empty array or object for a failed tool. The model may interpret it as a valid empty result.

### 2. Missing `is_error`

Do not provide only a prose failure message. Set the explicit error flag and include structured metadata.

### 3. Generic Error Messages

“Operation failed” does not tell the coordinator whether to retry, fix input, use a fallback, or escalate.

### 4. Retrying Every Error

Do not retry policy, permission, or unchanged validation failures. Retry only when the failure category and corrected conditions support it.

### 5. Escalating Every Failure Immediately

Let the current sub-agent attempt bounded local recovery for transient failures before involving the coordinator.

### 6. Terminating the Entire Pipeline on One Untried Failure

A transient error should not automatically destroy the workflow. Propagate a structured failure after local recovery options are exhausted.

### 7. Giving Every Agent Every Tool

More tools can reduce routing reliability and violate least privilege. Scope tools to the agent's role.

### 8. Confusing `auto` and `any`

`auto` allows the model to choose no tool. `any` requires at least one tool call.

## Exam Checklist

- Set `is_error: true` when a tool operation fails.
- Include an error category, retry decision, description, and recovery context.
- Retry transient failures with bounded attempts and appropriate delay.
- Correct validation input before retrying.
- Do not retry business-logic or permission failures without a changed condition.
- Let sub-agents attempt local transient recovery before escalating.
- Scope tools by agent role and follow least privilege.
- Remember that too many overlapping tools degrade selection reliability.
- Use `tool_choice: auto` for optional model-selected tool use.
- Use `tool_choice: any` when at least one tool call is mandatory but the model chooses which.
- Use `tool_choice: tool` when one exact tool must be called.
- Keep tool routing, policy enforcement, and error recovery as separate concerns.
- Never replace structured error signaling with an empty success-looking result.

## Final Summary

Reliable agents make failures visible and actionable. A failed tool should return `is_error: true` with a category, retry decision, explanation, and recovery context. Transient failures can receive bounded local retries; validation failures require corrected input; business-logic and permission failures should not be blindly retried.

For tool control, **scope each agent to relevant tools, use `auto` for optional tool use, `any` for mandatory use of one available tool, and `tool` for one exact required tool.**