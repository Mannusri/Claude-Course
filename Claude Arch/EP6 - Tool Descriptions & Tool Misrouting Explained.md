# EP06: Tool Descriptions and Tool Misrouting Explained

## Lesson Goal

This lesson explains how Claude selects tools and why unclear tool names, overlapping descriptions, and generic system prompts can cause misrouting.

The main exam topics are:

- How tool descriptions influence model routing
- The anatomy of a production-grade tool description
- Mutually exclusive tool boundaries
- Specific tool names and appropriate tool splitting
- System-prompt alignment
- Routing problems versus policy-enforcement problems
- Common exam distractors

## How Claude Selects a Tool

During an agentic loop, Claude considers the available tool definitions. The tool's name, description, and input schema provide the information Claude uses to decide whether and how to call it.

The description is not merely developer documentation. It is operational guidance for the model.

```text
User request
    -> Claude compares available tool definitions
    -> Claude selects the best matching tool
    -> Application validates and executes the request
```

There may be no separate routing layer that understands the business meaning of a tool. If several tools sound similar, Claude may choose the wrong one or fail to call the needed tool.

## Production-Grade Tool Descriptions

A useful description should answer four questions:

1. **What does the tool do?**
2. **When should Claude call it?**
3. **What should Claude not use it for?**
4. **What does it return?**

It can also state prerequisites, ordering constraints, side effects, and important boundaries.

### Weak Description

```text
Get customer information.
```

This does not clarify which customer data is returned, when to use the tool, or how it relates to other tools.

### Strong Description

```text
Retrieve a verified customer profile by customer ID.
Return the customer's name, email, plan tier, and account standing.
Call this first before any order lookup or refund operation because the
customer must be verified before downstream actions.
Do not use this tool to retrieve order history or process refunds.
```

The stronger description gives Claude functional purpose, timing, output, prerequisites, and negative boundaries.

## Description Anatomy

### Purpose

State the exact action and domain object.

```text
Retrieve a verified customer profile by customer ID.
```

### Invocation Criteria

State the user intent or workflow state that should trigger the tool.

```text
Call when the user asks about account identity, plan details, or account standing.
```

### Prerequisites and Ordering

State required earlier actions.

```text
Call this before order lookup or refund processing.
```

### Negative Boundaries

Explain when not to use the tool.

```text
Do not use this tool for order status, order history, or refunds.
```

### Return Contract

Describe the output Claude will receive.

```text
Returns customer ID, name, email, plan tier, verification status, and account standing.
```

### Side Effects

For tools that change state, describe the effect clearly.

```text
Issues a refund and changes the order's refund status. Use only after
customer verification, order ownership, and refund eligibility are confirmed.
```

## Customer Support Tool Examples

### `get_customer`

```text
Retrieve a verified customer profile by customer ID.
Call first when a support request requires customer identity or account
verification. Return name, email, plan tier, and account standing.
Do not use for order lookup, order history, or refund processing.
```

### `lookup_order`

```text
Look up one order by order ID and confirm its customer association.
Call after customer verification when the user asks about order status,
delivery, ownership, or refund eligibility. Return order status, customer ID,
amount, and relevant delivery details.
Do not use this tool to issue a refund or retrieve a customer's general profile.
```

### `process_refund`

```text
Issue a refund for an eligible order.
Call only after get_customer has verified the customer and lookup_order has
confirmed that the order exists, belongs to that customer, and is refundable.
Do not call for an unverified customer, an order owned by another customer,
or an order that fails the refund policy.
Return the refund status and transaction reference.
```

Descriptions improve routing, but they do not replace deterministic policy gates. A refund amount limit should still be enforced in application code or a pre-tool hook.

## Overlapping Descriptions and Misrouting

Tool misrouting often occurs when several descriptions use the same broad words.

### Ambiguous Tool Set

```text
search_web: Search for information.
search_documents: Search documents for information.
analyze_content: Analyze and extract information from content.
```

These descriptions overlap around “search,” “information,” and “content.” Claude may not know which tool owns a particular request.

### Mutually Exclusive Tool Set

```text
search_live_web:
  Query live internet pages for current events, recent publications, and
  URLs that are not already in the research corpus. Return ranked URLs and snippets.
  Do not use for documents already loaded into the research corpus.

search_research_corpus:
  Search the preloaded internal research corpus. Use only for documents already
  ingested into the application. Do not use for current internet information.

analyze_retrieved_content:
  Perform deep analysis of content that has already been retrieved by another
  tool. Do not use this tool to search the web or locate documents.
```

The boundaries are clear:

- Live web search
- Existing corpus search
- Post-retrieval analysis

## Tool Naming

Generic names have little semantic value:

- `search`
- `get_data`
- `fetch_info`
- `process`
- `handle_request`

These names force Claude to infer too much from vague labels.

Prefer names that identify the action and object:

- `search_live_web`
- `get_customer_by_id`
- `retrieve_order_history`
- `lookup_order`
- `process_refund`
- `analyze_retrieved_content`

Specific names make the tool inventory easier for Claude to distinguish and easier for developers to maintain.

## Tool Splitting

A generic tool that attempts to handle unrelated operations can create routing ambiguity. Split it into focused tools when the operations have different purposes, permissions, inputs, or side effects.

Instead of:

```text
get_data(resource, operation, filters, action)
```

Prefer focused tools such as:

```text
get_customer_by_id(customer_id)
list_customer_orders(customer_id)
lookup_order(order_id)
process_refund(order_id, amount)
```

This resembles a well-defined repository or data-access layer: each function has a clear responsibility.

## System Prompt Alignment

Tool descriptions and the system prompt should reinforce the same workflow.

A generic system prompt such as:

```text
You are a customer support assistant.
```

may bias Claude toward whichever tool sounds most generally relevant. A richer system prompt can define the role, goal, and normal sequence:

```text
You are a customer support assistant responsible for resolving returns and refunds.

For refund requests:
1. Call get_customer first to verify identity.
2. Call lookup_order to confirm the order and ownership.
3. Call process_refund only when the order is eligible.
4. Escalate to a human only when policy or missing information prevents resolution.
```

The system prompt supplies workflow context, while tool descriptions explain each tool's exact boundary and use.

### Avoid Conflicting Instructions

If the system prompt says that `process_refund` may be used at any time but the tool description says it requires verification, the model receives conflicting guidance. Align both layers and enforce critical constraints in code.

## Diagnose the Problem Before Choosing a Fix

Different symptoms require different solutions.

| Symptom | First thing to inspect |
| --- | --- |
| Wrong tool selected | Tool names and descriptions |
| Needed tool not selected | Invocation criteria and system prompt |
| Similar tools confused | Overlapping descriptions and missing boundaries |
| Tool executes when it should be blocked | Pre-tool hook or application policy |
| Raw output is hard to read | Post-tool hook |
| Tool order is violated | Prerequisite gate or session-state check |
| Final handoff lacks context | Structured handoff summary |

Do not use a hook to solve a description problem unless the requirement is a deterministic policy. Do not use prompt examples as the primary solution for a missing authorization gate.

## Exam Scenarios

### Scenario 1: Auto-Resolvable Cases Are Escalated

**Question:** A support agent frequently escalates refunds that it should resolve automatically. The tools have one-sentence descriptions. What should be inspected first?

**Answer:** Improve the tool descriptions so they clearly state when to call the tool, what it returns, prerequisites, and when not to use it. Check that the system prompt aligns with the workflow.

### Scenario 2: Similar Research Tools

**Question:** A coordinator has tools for web search, document search, and content analysis, but frequently chooses the wrong tool. What is the best fix?

**Answer:** Rewrite the descriptions with mutually exclusive scopes: live web, already-ingested corpus, and post-retrieval analysis.

### Scenario 3: Refund Policy Violation

**Question:** The agent sometimes processes refunds above the authorization limit despite a clear description. What should guarantee the limit?

**Answer:** A deterministic pre-tool policy gate or application-level validation. Tool descriptions improve routing but cannot guarantee compliance.

### Scenario 4: Generic Tool Names

**Question:** Claude has trouble distinguishing `search`, `get_data`, and `process`. What should change?

**Answer:** Rename or split the tools into specific, semantically meaningful operations such as `lookup_order` and `process_refund`.

### Scenario 5: Missing Prerequisite

**Question:** Claude calls `process_refund` without verifying the customer or checking the order. What should the description and system prompt communicate?

**Answer:** Both should state the required sequence and that the refund tool must not be used until verification and order-ownership checks succeed. Add a prerequisite gate for deterministic enforcement.

### Scenario 6: Retrieved Content Analysis

**Question:** A tool should analyze content already retrieved by another tool, not search for new content. What description is clearest?

**Answer:** State that it performs post-retrieval analysis and explicitly exclude web search and document retrieval.

## Exam Anti-Patterns

### 1. Treating Descriptions as Developer-Only Documentation

Descriptions are part of the model's routing context. Write them for Claude as well as for human maintainers.

### 2. Reusing the Same Broad Vocabulary

Descriptions that all say “search for information” create overlap. Define mutually exclusive data sources and stages.

### 3. Using Few-Shot Examples as the Main Routing Fix

Examples may improve consistency, but ambiguous tool definitions remain ambiguous. Fix names, descriptions, boundaries, and schemas first.

### 4. Using Prompts to Enforce Critical Policy

A description or system prompt cannot provide a hard guarantee. Use pre-tool hooks and application validation for non-negotiable rules.

### 5. One Generic Tool for Everything

Overly broad tools have unclear semantics and permissions. Split unrelated operations into focused tools.

### 6. Misaligned System Prompt

A generic or conflicting system role can bias Claude toward the wrong tool category. Make the role and workflow explicit.

## Exam Checklist

- Remember that tool descriptions influence Claude's routing decisions.
- Describe what the tool does, when to call it, when not to call it, and what it returns.
- Include prerequisites, ordering, side effects, and input expectations where relevant.
- Make overlapping tools mutually exclusive by domain, source, or workflow stage.
- Prefer specific semantic names over generic names such as `search` or `process`.
- Split tools with unrelated responsibilities or permissions.
- Align the system prompt with the tool descriptions and intended sequence.
- Use hooks for deterministic policy enforcement, not as the first fix for ambiguous routing.
- Use `PostToolUse` for result normalization, not tool selection.
- Use prerequisite gates when tool ordering must be guaranteed.
- Treat few-shot examples as a consistency aid, not a compliance guarantee.
- Inspect descriptions and naming before changing model or adding unnecessary prompts.

## Final Summary

Claude routes tool calls using the semantic information in the available tool definitions and surrounding instructions. Production-grade tools need clear purpose, invocation criteria, negative boundaries, return contracts, and prerequisites.

For the exam, remember: **fix tool misrouting with precise names and mutually exclusive descriptions; fix non-negotiable policy violations with deterministic hooks or application code.**