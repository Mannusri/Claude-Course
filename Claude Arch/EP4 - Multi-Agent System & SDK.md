# EP4: Multi-Agent System and Claude SDK

## Lesson Goal

This capstone lesson combines the concepts from the previous episodes into a practical multi-agent workflow implemented with Python and the Anthropic SDK.

The example handles a customer refund request:

```text
Customer C001 wants a refund for order 0100.
```

The coordinator delegates the work to specialized sub-agents, passes structured findings between them, aggregates the results, and returns a final case summary.

The main exam topics are:

- Coordinator decomposition
- Customer-verification and refund-processing sub-agents
- Structured context passing
- Tool schemas and tool registries
- Independent agentic loops
- Conditional delegation and failure handling
- The difference between raw SDK orchestration and a `task` tool

## Capstone Architecture

```text
Customer refund request
            |
            v
      Coordinator
            |
            v
 Customer verification agent
            |
    Structured finding
            |
            v
 Refund processor agent
       /            \
 Lookup order     Process refund
            |
            v
      Coordinator aggregation
            |
            v
       Final summary
```

### Coordinator

The coordinator:

1. Decomposes the refund request.
2. Runs customer verification first.
3. Passes the verification finding to the refund processor.
4. Prevents downstream work when verification fails.
5. Aggregates the sub-agent results.
6. Returns the final customer-support summary.

### Customer Verification Agent

This sub-agent verifies whether the customer exists and is eligible to continue the workflow.

Its primary tool is:

```text
get_customer(customer_id)
```

### Refund Processor Agent

This sub-agent uses the verified customer information to:

1. Look up the order.
2. Confirm that the order belongs to the customer.
3. Determine whether the order is eligible for a refund.
4. Process the refund when the eligibility checks succeed.

Its tools are:

```text
lookup_order(order_id)
process_refund(order_id)
```

## Tool Schemas

The model needs tool metadata so it can identify which tool to use and provide valid input. A tool definition normally contains a name, description, and input schema.

Example schemas:

```json
[
  {
    "name": "get_customer",
    "description": "Look up a customer by customer ID.",
    "input_schema": {
      "type": "object",
      "properties": {
        "customer_id": {
          "type": "string",
          "description": "The customer identifier"
        }
      },
      "required": ["customer_id"]
    }
  },
  {
    "name": "lookup_order",
    "description": "Look up an order by order ID and verify its owner.",
    "input_schema": {
      "type": "object",
      "properties": {
        "order_id": {
          "type": "string",
          "description": "The order identifier"
        }
      },
      "required": ["order_id"]
    }
  },
  {
    "name": "process_refund",
    "description": "Issue a refund for an eligible order.",
    "input_schema": {
      "type": "object",
      "properties": {
        "order_id": {
          "type": "string",
          "description": "The order identifier"
        }
      },
      "required": ["order_id"]
    }
  }
]
```

Tool definitions tell Claude what is available. The Python application owns the actual functions and must validate tool names and arguments before execution.

## Tool Registry

A tool registry maps the model-visible tool name to the application function that executes it.

Conceptually:

```python
tool_registry = {
    "get_customer": get_customer,
    "lookup_order": lookup_order,
    "process_refund": process_refund,
}
```

The registry provides a controlled boundary between model requests and application code. The model cannot directly execute Python functions. It requests a tool, and the application resolves the name through the registry, validates the input, executes the function, and returns a tool result.

## Running a Sub-Agent

A sub-agent is still an agent. It runs its own agentic loop with:

- Its own prompt
- Its own message history
- Its own allowed tools
- Its own tool execution
- Its own `stop_reason` handling

The fact that an agent was spawned by a coordinator does not remove the need for the standard agent loop.

Conceptually:

```text
run_sub_agent(prompt, tools)
    -> send messages and tools to Claude
    -> inspect stop_reason
    -> end_turn: return final text
    -> tool_use: execute tools and continue
```

For every `tool_use` response:

1. Append the complete assistant response.
2. Execute the requested tool through the registry.
3. Create tool-result blocks with matching tool-use IDs.
4. Append the results as the next user message.
5. Call Claude again with the full history.

The loop ends normally only when `stop_reason` is `end_turn`.

## Structured Context Passing

The coordinator must pass the first sub-agent's result explicitly to the second sub-agent. Do not rely on the second agent seeing the first agent's conversation.

Example typed finding:

```python
verification_finding = {
    "customer_id": "C001",
    "exists": True,
    "customer_name": "Alice",
    "verification_status": "verified",
    "source": "customer_lookup",
}
```

The refund processor prompt should include the actual structured finding:

```text
Process the refund request for order 0100.

Prior customer verification:
{
  "customer_id": "C001",
  "exists": true,
  "customer_name": "Alice",
  "verification_status": "verified"
}

Confirm the order belongs to this customer before processing a refund.
```

The important properties are explicit fields, clear status values, and a direct handoff from one stage to the next.

## Conditional Delegation

The coordinator should not automatically invoke every sub-agent. It should inspect the verification result first.

### Successful Verification

```text
Customer exists and is verified
    -> Invoke refund processor
    -> Look up order
    -> Check ownership and eligibility
    -> Process refund if valid
```

### Failed Verification

```text
Customer does not exist
    -> Stop the workflow
    -> Do not invoke refund processor
    -> Return a clear failure summary
```

Example failure result:

```text
Customer support case resolved: customer C99 does not exist.
The customer lookup returned no matching record, so the refund processor was not invoked.
```

This prevents downstream agents from hallucinating customer, order, or refund information.

## Example Execution

### Successful Case

Input:

```text
Customer C001 wants a refund for order 0100.
```

Execution:

1. The coordinator decomposes the request.
2. The verification agent calls `get_customer` for `C001`.
3. The verification agent returns a structured finding confirming the customer.
4. The coordinator injects that finding into the refund processor prompt.
5. The refund processor calls `lookup_order` for `0100`.
6. The processor confirms ownership and eligibility.
7. The processor calls `process_refund`.
8. The coordinator aggregates the verification and refund results.
9. The coordinator returns the final summary.

### Failed Case

Input:

```text
Customer C99 wants a refund for order 0100.
```

Execution:

1. The coordinator calls the verification agent.
2. `get_customer` returns no matching customer.
3. The verification agent returns `exists: false`.
4. The coordinator stops the workflow.
5. The refund processor is not invoked.
6. The coordinator returns the verification failure.

## Raw Anthropic SDK Versus `task` Tool

This distinction is important for the exam.

### Raw Anthropic Python SDK

The raw Anthropic Python SDK provides the API client and message/tool interfaces. It does not automatically provide a built-in `task` tool for spawning sub-agents.

A developer can still implement the multi-agent architecture by writing ordinary Python orchestration functions:

```text
coordinator function
    -> calls run_sub_agent for verification
    -> passes structured result
    -> calls run_sub_agent for refund processing
    -> aggregates results
```

This is still a valid multi-agent design because the coordinator and sub-agents have separate roles, prompts, tools, and loops.

### Claude Code or Agent SDK Capability

In environments that provide a built-in `task` tool, the coordinator can use that tool to spawn sub-agents declaratively. The coordinator must have `task` in its allowed tools.

### Exam Distinction

Do not conclude that a design is invalid merely because a raw Python SDK example uses ordinary functions instead of a `task` call. Understand the architectural pattern separately from the platform-specific spawning mechanism.

| Capability | Raw Anthropic Python SDK | Claude Code or agent SDK |
| --- | --- | --- |
| Call Claude models | Yes | Yes |
| Define and execute application tools | Yes, through application code | Yes |
| Built-in `task` sub-agent spawning | Not automatically available | Available when supported by the environment |
| Manual coordinator orchestration | Yes | Yes |

## Safety and Validation Boundaries

Refund processing is a consequential action. The application should validate each boundary rather than trusting model-generated arguments.

Validate:

- Customer ID format
- Customer existence
- Order ID format
- Order ownership
- Order status
- Refund eligibility
- Duplicate-refund protection
- Tool name and input schema
- Authorization to issue the refund

The model can recommend or request an action, but application code should enforce business rules before executing it.

## Exam Anti-Patterns

### 1. Passing an Unstructured String

Do not pass vague text such as “the previous agent verified the customer.” Pass explicit fields such as `customer_id`, `exists`, and `verification_status`.

### 2. Invoking Downstream Work After Failed Verification

Do not run the refund processor when the customer lookup failed. Stop or return for clarification.

### 3. Assuming Sub-Agents Share Context

Every sub-agent has its own history. Inject prior findings into the next agent's prompt.

### 4. Treating Tool Definitions as Implementations

A schema only describes a tool to Claude. The application still needs a registry and a real function implementation.

### 5. Parsing Text Instead of `stop_reason`

Use structured stop state to control each agentic loop. Do not stop because the response text contains words such as `done` or `successful`.

### 6. Confusing SDK Layers

Do not assume the raw Anthropic Python SDK automatically supports a built-in `task` tool. Check which SDK or runtime is being used.

## Exam Scenarios

### Scenario 1: Missing Customer

**Question:** The customer-verification agent returns `exists: false`. What should the coordinator do?

**Answer:** Stop the workflow, do not invoke the refund processor, and return a clear verification failure.

### Scenario 2: Context Handoff

**Question:** The refund processor needs the verification result. How should the coordinator provide it?

**Answer:** Pass a structured finding with explicit fields in the processor's task prompt or input.

### Scenario 3: Tool Execution

**Question:** Claude requests `process_refund`. Does Claude execute the Python function?

**Answer:** No. The application validates the request, resolves the function through the tool registry, executes it, and returns the tool result.

### Scenario 4: Loop Completion

**Question:** A sub-agent returns `stop_reason: "tool_use"`. What happens next?

**Answer:** Append the assistant response, execute the requested tool, append the matching tool result, and continue the loop.

### Scenario 5: SDK Capability

**Question:** A Python implementation uses coordinator functions rather than a built-in `task` tool. Is it necessarily incorrect?

**Answer:** No. Manual orchestration is valid with the raw SDK. The built-in `task` tool is a platform or agent-SDK capability, not automatically part of the raw Anthropic Python SDK.

### Scenario 6: Refund Safety

**Question:** The model requests a refund for an order but the order belongs to a different customer. What should happen?

**Answer:** The application must reject the action. Model output does not override ownership and business-rule validation.

## Exam Checklist

- Know the coordinator's decomposition and aggregation responsibilities.
- Understand that every sub-agent runs its own agentic loop.
- Pass structured findings between sub-agents.
- Define tools with clear names, descriptions, and input schemas.
- Use a tool registry to map model requests to application functions.
- Append full assistant tool-use messages before tool results.
- Continue on `tool_use` and exit on `end_turn`.
- Do not invoke downstream agents after a failed prerequisite.
- Do not assume sub-agents share conversation history.
- Distinguish tool metadata from actual tool implementation.
- Understand manual orchestration with the raw Anthropic Python SDK.
- Know that built-in `task` spawning depends on Claude Code or an agent SDK that supports it.
- Validate consequential actions in application code.
- Use structured status fields to make handoffs and failure paths explicit.

## Final Summary

The capstone workflow demonstrates how a coordinator can combine specialized agents into a reliable customer-support process. The coordinator verifies the customer, passes a structured finding to the refund processor, allows the processor to use only its assigned tools, and aggregates the final result.

For the exam, remember the key distinction: **multi-agent architecture is a design pattern, while `task` is a platform-specific spawning mechanism. With the raw Anthropic Python SDK, the same architecture can be implemented through explicit Python orchestration and independent sub-agent loops.**