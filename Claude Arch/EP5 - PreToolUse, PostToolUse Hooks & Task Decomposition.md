# EP05: PreToolUse, PostToolUse Hooks and Task Decomposition

## Lesson Goal

This lesson explains how to enforce critical policies around tool calls and how to choose the right task-decomposition strategy for an agentic workflow.

The main exam topics are:

- Probabilistic prompts versus deterministic hooks
- `PreToolUse` policy enforcement
- `PostToolUse` result processing
- Prerequisite gates and session state
- Structured handoff summaries
- Prompt chaining
- Dynamic decomposition
- Common distractor answers and anti-patterns

## Prompts Versus Programmatic Enforcement

Language-model instructions are probabilistic. A system prompt can strongly request a rule, but the model may still interpret or apply that instruction incorrectly.

Example prompt:

```text
Only process refunds under $500.
```

This is useful guidance, but it is not a guaranteed control. If the requirement is business-critical, the application must enforce it in code.

### When Enforcement Must Be Deterministic

Exam questions often signal this requirement with phrases such as:

- Must never happen
- Must be guaranteed
- Policy requires
- Cannot be overridden
- Business-critical operation
- Must block the action

For these requirements, choose programmatic enforcement, such as a hook or a prerequisite gate, rather than a stronger prompt.

| Mechanism | Behavior | Suitable for guarantees? |
| --- | --- | --- |
| Prompt instruction | Probabilistic model guidance | No |
| Few-shot examples | Improves consistency and format | No |
| Pre-tool hook | Deterministic gate before execution | Yes |
| Post-tool hook | Deterministic processing after execution | Yes, for result handling |
| Application validation | Enforces business rules in code | Yes |

## Hook Lifecycle

Hooks are code interceptors placed at defined points in the agentic tool flow.

```text
Claude requests a tool
        |
        v
   PreToolUse hook
        |
   allowed? ---- no ----> Block, reject, or escalate
        |
       yes
        v
   Execute the tool
        |
        v
   PostToolUse hook
        |
        v
   Append processed result and continue the loop
```

## PreToolUse Hooks

A `PreToolUse` hook runs after Claude requests a tool but before the application executes it.

Use it to:

- Enforce authorization rules
- Validate arguments
- Check prerequisites
- Enforce amount or scope limits
- Block unsafe operations
- Route an action to human review
- Reject malformed or unauthorized requests

### Refund Policy Example

Suppose an agent can call `process_refund`, but refunds above $500 require senior support approval.

The pre-tool policy should evaluate the request before execution:

```python
def enforce_refund_policy(tool_name, tool_params):
    if tool_name != "process_refund":
        return {"allowed": True}

    if tool_params["amount"] > 500:
        return {
            "allowed": False,
            "reason": "Refund exceeds the agent authorization limit.",
            "action": "escalate_to_human",
            "routing_tier": "senior_support",
        }

    return {"allowed": True}
```

The refund function must not run when the hook returns `allowed: false`.

### Pass-Through Behavior

A hook should pass through tools it does not control. For example, a refund-policy hook should return an allowed result for `get_customer` or `lookup_order` unless it is explicitly responsible for those tools.

This keeps hooks focused and prevents unrelated tools from being blocked accidentally.

## PostToolUse Hooks

A `PostToolUse` hook runs after the tool has executed and returned a raw result.

Use it to:

- Normalize database status codes
- Convert timestamps into readable dates
- Remove internal fields
- Validate or enrich returned data
- Apply a consistent output schema
- Add metadata for downstream agents
- Update workflow state after a successful tool call

### Normalization Example

A database may return:

```json
{
  "status": 3,
  "estimated_delivery": 1735689600
}
```

The `PostToolUse` hook can convert it into a model- and user-friendly result:

```json
{
  "status": "shipped",
  "estimated_delivery": "2025-01-01"
}
```

The model receives the normalized result and can produce a clear final answer.

### Post-Tool Registry

Applications can maintain a registry of post-tool hooks:

```python
post_tool_hooks = [
    normalize_order_result,
    redact_internal_fields,
]
```

Each hook should identify whether it applies to the current tool. A hook that does not apply should return the result unchanged.

## Pre-Tool Versus Post-Tool Exam Rule

| Requirement | Correct hook |
| --- | --- |
| Block a refund above a policy limit | `PreToolUse` |
| Verify authorization before deletion | `PreToolUse` |
| Ensure a customer is verified first | `PreToolUse` prerequisite gate |
| Map database status codes to labels | `PostToolUse` |
| Convert epoch timestamps to dates | `PostToolUse` |
| Normalize raw tool output | `PostToolUse` |
| Create a structured escalation after a block | `PreToolUse` result plus handoff summary |

If the action must be stopped, the check must happen before the tool runs. Do not execute first and validate afterward.

## Prerequisite Gates

A prerequisite gate is a `PreToolUse` hook that checks workflow state before allowing a downstream tool to run.

### Customer-Verification Example

The workflow requires:

```text
get_customer -> lookup_order -> process_refund
```

The refund-related tools must not run until customer verification succeeds.

```python
session_state = {
    "customer_verified": False,
}

def require_verified_customer(tool_name, session_state):
    protected_tools = {"lookup_order", "process_refund"}

    if tool_name in protected_tools and not session_state["customer_verified"]:
        return {
            "allowed": False,
            "reason": "Customer verification is required first.",
        }

    return {"allowed": True}
```

After `get_customer` succeeds, application code or a post-tool hook updates the state:

```python
session_state["customer_verified"] = True
```

The next pre-tool check can then allow `lookup_order` and `process_refund`.

### Why a Prompt Is Not Enough

The instruction “always verify the customer before processing a refund” may be followed most of the time, but it does not guarantee ordering. A prerequisite gate enforces the dependency regardless of the model's decision.

## Structured Handoff Summaries

When an action is blocked or escalated, a short message such as “please assist” is not enough. The receiving human or agent should not have to reconstruct the entire conversation.

A handoff summary should be self-contained and include:

- Case or request ID
- Customer identity and verification status
- Order ID
- Requested action
- Tool or policy that blocked the action
- Relevant amount or risk
- Evidence and prior results
- Current status
- Required next action
- Priority and routing tier

Example:

```json
{
  "case_id": "CASE-0100",
  "customer_id": "C001",
  "customer_verified": true,
  "order_id": "O102",
  "requested_action": "process_refund",
  "amount": 750,
  "status": "blocked_by_policy",
  "reason": "Refund exceeds the agent authorization limit of 500.",
  "next_action": "Review and approve or reject the refund.",
  "priority": "high",
  "routing_tier": "senior_support"
}
```

### Exam Rule

When escalating to a human or another agent, choose a **structured, self-contained handoff summary** rather than a vague message or a request to reread the full transcript.

## Task Decomposition Strategies

Before choosing a workflow, ask:

```text
Are the steps known before execution begins?
```

- If yes, use prompt chaining.
- If no, and the next step depends on discoveries, use dynamic decomposition.

## Prompt Chaining

Prompt chaining divides a task into a predefined sequence. The output of one step becomes the input to the next step.

```text
Step 1: Review each changed file
    -> Step 2: Identify cross-file concerns
    -> Step 3: Rank issues by severity
    -> Step 4: Produce the final review
```

### Characteristics

- Steps are known in advance.
- Execution order is fixed.
- Each step has a focused responsibility.
- Outputs flow directly to the next step.
- The workflow is predictable and reproducible.

### Good Use Cases

- Pull-request review pipelines
- Compliance checks
- CI/CD validation
- Fixed document-processing workflows
- Known-format transformations

### Limitation

Do not use prompt chaining as the primary strategy for open-ended investigations. A fixed sequence can become wrong when findings diverge from the assumptions made at design time.

## Dynamic Decomposition

Dynamic decomposition allows the agent or coordinator to decide the next step based on findings gathered during execution.

Example:

```text
Investigate the production incident.
Start with error logs from the last 24 hours.
Based on what you find, choose the next investigation step.
Continue until you identify the root cause and remediation.
```

The available tools might include:

- `read_logs`
- `query_database`
- `check_configuration`
- `trace_request`

### Characteristics

- The exact sequence is not known in advance.
- The next action depends on current findings.
- The workflow adapts as evidence changes.
- The coordinator or agent decomposes the task dynamically.

### Good Use Cases

- Production incident investigation
- Open-ended research
- Unknown bug diagnosis
- Complex migrations with uncertain dependencies
- Exploratory analysis

## Prompt Chaining Versus Dynamic Decomposition

| Dimension | Prompt chaining | Dynamic decomposition |
| --- | --- | --- |
| Steps | Defined up front | Generated from findings |
| Control | High and predictable | Flexible and adaptive |
| Best for | Reproducible workflows | Open-ended investigation |
| Example | CI/CD review pipeline | Production incident diagnosis |
| Main risk | Fixed steps become invalid | Less predictable cost and path |

## Common Exam Distractors

### “Use a Stronger Prompt”

Reject this when the requirement is a guarantee. Stronger wording remains probabilistic. Use programmatic enforcement.

### “Add Few-Shot Examples”

Few-shot examples improve consistency and output style, but they do not guarantee that a critical tool call will be blocked.

### “Check After Execution”

Reject this when the tool must not run. A post-tool check cannot undo a dangerous side effect that has already occurred.

### “Use a Fixed Sequence for an Unknown Investigation”

Reject prompt chaining when the next action depends on discoveries that are not yet known. Use dynamic decomposition.

### “Escalate with a Short Message”

Reject vague handoffs. Include the context, evidence, reason, and next action in a structured summary.

## Exam Scenarios

### Scenario 1: Refund Limit

**Question:** A refund above $500 must never be processed automatically. Which mechanism provides the strongest guarantee?

**Answer:** A `PreToolUse` hook or application-level policy gate that blocks `process_refund` before execution.

### Scenario 2: Raw Database Result

**Question:** The order tool returns numeric status codes and epoch timestamps, but the model needs readable values. Where should conversion happen?

**Answer:** In a `PostToolUse` hook after the tool returns and before the result is sent back to the model.

### Scenario 3: Refund Before Verification

**Question:** The model sometimes calls `process_refund` before `get_customer`. What reliably enforces the correct order?

**Answer:** A prerequisite gate implemented as a `PreToolUse` hook that checks a `customer_verified` session flag.

### Scenario 4: Escalation

**Question:** A refund is blocked and sent to senior support. What should the agent provide?

**Answer:** A structured, self-contained handoff summary containing customer, order, amount, reason, evidence, status, routing, and required action.

### Scenario 5: Fixed Code Review

**Question:** A workflow always reviews changed files, checks cross-file concerns, ranks issues, and writes a summary. Which decomposition strategy fits?

**Answer:** Prompt chaining, because the steps and order are known in advance.

### Scenario 6: Production Incident

**Question:** An agent must inspect logs and choose subsequent actions based on what it discovers. Which strategy fits?

**Answer:** Dynamic decomposition, because the next step cannot be known before the findings are available.

### Scenario 7: Prompt Cannot Guarantee Policy

**Question:** A system prompt says “never delete production data,” but the operation is business-critical. What should be added?

**Answer:** A deterministic pre-tool policy gate with authorization and environment checks.

## Exam Checklist

- Treat prompts as probabilistic guidance.
- Use code and hooks for guaranteed business rules.
- Use `PreToolUse` to inspect, authorize, block, or gate before execution.
- Use `PostToolUse` to normalize, enrich, or transform returned data.
- Keep hooks pass-through for tools they do not control.
- Implement prerequisite ordering with session-state gates.
- Set verification state only after the prerequisite tool succeeds.
- Never rely on the model alone for critical sequencing.
- Escalate with a structured, self-contained handoff summary.
- Use prompt chaining when steps are known up front.
- Use dynamic decomposition when the next step depends on discoveries.
- Do not use fixed chains for open-ended investigations.
- Do not assume few-shot examples guarantee compliance.
- Do not validate after execution when the action must be blocked before it happens.

## Final Summary

Prompts guide model behavior, but hooks and application code enforce deterministic policy. `PreToolUse` is the control point for authorization, prerequisites, and blocking unsafe actions. `PostToolUse` is the control point for normalizing and enriching tool results.

For task decomposition, **use prompt chaining for known, reproducible sequences and dynamic decomposition for open-ended workflows whose next steps depend on findings.**