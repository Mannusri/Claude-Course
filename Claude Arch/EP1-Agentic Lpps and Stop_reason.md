# EP1: Agentic Loops and Stop Reasons

## Lesson Goal

This lesson explains the execution engine behind a Claude agent and the concepts most likely to matter in exam scenarios:

- How an agent reasons, acts, observes, and decides
- How `stop_reason` controls the agentic loop
- How Claude tools are declared and executed
- Why the complete conversation history must be sent on every API request
- How `system`, `user`, and `assistant` roles work
- Which implementation patterns to use and which anti-patterns to avoid

## Hands-on Setup

Hands-on work with the Claude SDK is optional for exam preparation, but it helps connect the theory to implementation.

### Requirements

1. Create an account at [platform.claude.com](https://platform.claude.com).
2. Add credits if you want to make API calls. A small balance may be enough for the examples in this course.
3. Create an Anthropic API key.
4. Store the key securely in the `ANTHROPIC_API_KEY` environment variable.
5. Use a `.env` file with appropriate secret-handling practices if environment-based configuration is preferred.

Never commit an API key to source control or place it directly in application code.

Example environment-variable configuration:

```text
ANTHROPIC_API_KEY=your-api-key
```

The SDK reads this variable when creating the Anthropic client. A lightweight Claude model can be used for practice to reduce cost, but always verify the current model name in the official documentation before running code.

## Message Roles

Every message sent through the Claude API is associated with a role.

### System Role

The system instruction defines the assistant's identity, behavior, constraints, or operating context. For example:

```text
You are a social media marketer. Follow the campaign rules provided by the user.
```

It establishes how the model should behave across the interaction.

### User Role

The user role represents input from the end user. This includes the initial request and tool results sent back to Claude by the application.

Examples:

- `Where is my order 4821?`
- A tool result returned by the application after executing a requested tool

### Assistant Role

The assistant role represents Claude's response. When Claude requests a tool, the application must preserve Claude's response as an assistant message before adding the tool result.

## Chat Versus Agentic Execution

In a normal chat interaction:

```text
User message -> Claude response -> Done
```

An agent uses a loop:

```text
User request
    -> Claude reasons
    -> Claude requests a tool
    -> Application executes the tool
    -> Application sends the result to Claude
    -> Claude reasons again
    -> Final response or another tool request
```

The model is the agent's reasoning engine. Tools are the actions available to the agent. Claude does not execute application functions directly. It requests a tool call; the application validates the request, executes the corresponding function, and returns the result.

## The Agentic Loop

The core lifecycle is:

1. Receive the user's request.
2. Add the request to the conversation history with the `user` role.
3. Call the Claude API with the current messages and available tool definitions.
4. Inspect `response.stop_reason`.
5. If the reason is `tool_use`, preserve Claude's response as an `assistant` message.
6. Execute each requested tool in the application.
7. Add the tool results as a `user` message.
8. Send the complete updated history back to Claude.
9. Repeat until the stop reason is `end_turn`.
10. Extract the final text and return it to the caller.

Conceptually:

```text
while true:
    response = call_claude(messages, tools)

    if response.stop_reason == "end_turn":
        return response text

    if response.stop_reason == "tool_use":
        append response.content as assistant message
        execute requested tools
        append tool results as user message
        continue
```

The loop should also have a maximum-iteration safety cap. The cap protects the application from an unexpected infinite loop; it is not the normal completion condition.

## Stop Reasons

### `tool_use`

`tool_use` means Claude wants the application to perform one or more tool calls. The response contains tool-use blocks that identify the requested tool and its input.

The application should:

1. Append the complete Claude response as an `assistant` message.
2. Read each tool name, tool-use ID, and input.
3. Validate the input.
4. Execute the matching application function.
5. Return a tool-result block containing the matching tool-use ID.
6. Append the tool results as a `user` message.
7. Call Claude again with the full history.

### `end_turn`

`end_turn` means Claude has finished the current turn. This is the normal primary exit from the agentic loop. The application should stop looping and extract any text blocks for the final answer.

For exam questions, remember:

| Stop reason | Meaning | Application action |
| --- | --- | --- |
| `tool_use` | Claude wants the application to act | Execute tools, append results, continue |
| `end_turn` | Claude has finished | Stop and return the final response |

The API may expose additional stop reasons for other conditions, such as token limits or configured stop sequences. The implementation must inspect the API's structured `stop_reason` value rather than assuming every response has only one possible state. For the agentic tool loop, `tool_use` and `end_turn` are the key control-flow values.

## Tool Definitions

A tool definition should clearly describe:

- `name`: The identifier Claude uses when requesting the tool
- `description`: When and why Claude should use it
- `input_schema`: The JSON schema for valid parameters

Example shape:

```json
{
  "name": "lookup_order",
  "description": "Look up an order by ID and return its status and delivery details.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {
        "type": "string",
        "description": "The numeric order ID"
      }
    },
    "required": ["order_id"]
  }
}
```

Claude uses the name, description, and schema to decide whether a tool is appropriate and what input to provide. The application owns the actual function implementation.

## Stateless API and Conversation History

The Claude API is stateless. It does not automatically remember a previous API request.

The application must send the full relevant conversation history on every request. For a tool call, the history generally grows like this:

```text
user: original request
assistant: tool_use block
user: tool_result block
assistant: final response
```

The order matters. When Claude requests a tool, append the assistant message first. Then append the tool result as the user message. The tool-use ID in the result must exactly match the ID in Claude's request.

## Exam-Focused Anti-Patterns

### 1. Parsing Natural Language to End the Loop

Do not stop because Claude generated text such as `I am finished` or `Task completed`.

Why it fails:

- Wording varies between responses.
- A tool-use turn may contain no text block.
- Natural-language parsing is fragile and not the API's control signal.

Use `response.stop_reason == "end_turn"` instead.

### 2. Using the Iteration Cap as the Completion Condition

An iteration limit is a safety valve, not the primary way to determine that the task is complete. Stop normally when the structured stop reason indicates `end_turn`; fail safely when the configured cap is reached.

### 3. Inspecting Content Type Instead of Control State

Do not end the loop merely because the response contains a text block, JSON, Boolean, or another content type. Content type describes the payload. `stop_reason` describes what the application should do next.

### 4. Hard-Coding a Rigid Tool Sequence

Do not assume that Claude will always call tools in one fixed order. The model should drive tool selection based on the request, available tools, and returned results.

### 5. Executing Tools Without Validation

Validate the requested tool name and input before executing application code. Handle unknown tools and invalid arguments explicitly.

## Exam Scenarios to Practice

### Scenario 1: Claude Requests a Tool

**Question:** Claude returns `stop_reason: "tool_use"`. What should the application do?

**Answer:** Append Claude's response as an assistant message, execute the requested tool, append the matching tool result as a user message, and call Claude again with the complete history.

### Scenario 2: Claude Finishes

**Question:** Claude returns `stop_reason: "end_turn"`. What should the application do?

**Answer:** Break the agentic loop and return the response's text to the caller.

### Scenario 3: Missing History

**Question:** A second API request contains only the latest tool result. What is wrong?

**Answer:** The API is stateless, so the application has lost the context. Send the complete conversation history, including the original user request, assistant tool-use message, and tool result.

### Scenario 4: Tool-Use ID Mismatch

**Question:** The application returns a tool result with a different ID from the requested tool-use block. What is the risk?

**Answer:** Claude cannot reliably associate the result with its request. Preserve and return the exact tool-use ID.

### Scenario 5: Fragile Completion Check

**Question:** Why is checking whether the response text contains `completed` a bad way to stop the loop?

**Answer:** Natural-language wording is not a reliable control signal. Use the structured `stop_reason` field.

## Exam Checklist

- Know the difference between system, user, and assistant roles.
- Remember that the Claude API is stateless.
- Send the complete conversation history on every request.
- Know that Claude requests tools but the application executes them.
- Inspect `stop_reason`, not generated wording.
- Treat `tool_use` as execute-and-continue.
- Treat `end_turn` as the normal loop exit.
- Append the assistant tool-use response before the user tool-result message.
- Preserve exact tool-use IDs.
- Use an iteration limit as a safety cap only.
- Validate tool names and tool inputs.
- Understand the difference between model-driven tool selection and a fixed workflow.

## Final Summary

An agentic loop is a controlled conversation between the application and Claude. Claude reasons and requests an action, the application executes the requested tool, and the result is sent back as part of the conversation history. The loop continues while `stop_reason` is `tool_use` and ends when Claude returns `end_turn`.

For the exam, the most important rule is simple: **use the structured stop reason to control the loop, preserve the complete history, and never replace API state with guesses based on generated text.**

## Architect's Cheat Sheet

### The 5 Golden Rules

1. **`end_turn` is the only valid normal loop exit.** Stop the agentic loop and return the final response.
2. **`tool_use` means execute and continue.** Run the requested tool, append its result, and call Claude again.
3. **Always append the assistant message first, then the user tool result.** This preserves the correct conversation order.
4. **`tool_use_id` must perfectly match.** The ID in the tool result must match the ID Claude provided in the tool-use block.
5. **The API is stateless.** Send the full conversation history on every request.

### The 3 Anti-Patterns to Spot on Exams

| Anti-pattern | Why it fails |
| --- | --- |
| Parsing words from Claude's response | Natural-language wording varies and can cause crashes or incorrect loop termination. |
| Using the iteration cap as the primary exit | The agent may stop before the task is complete. Use the cap only as a safety limit. |
| Checking text content type to decide whether to stop | A text block does not mean the task is complete; this can silently drop tool calls. |

### Exam Recognition Tip

When a scenario mentions missing history, a mismatched tool ID, or a hard-coded stop condition, it is testing one of these rules. Choose the solution that relies on structured API state, preserves message order, and continues the loop after `tool_use`.