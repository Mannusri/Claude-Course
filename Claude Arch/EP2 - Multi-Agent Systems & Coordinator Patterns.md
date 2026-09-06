# EP2: Multi-Agent Systems & Coordinator Patterns

## Lesson Goal

This lesson explains why a single agent may not be enough for complex work and how to design a multi-agent system with a coordinator and specialized sub-agents.

The main exam topics are:

- Single-agent limitations
- Hub-and-spoke architecture
- Coordinator responsibilities
- The `task` tool and allowed-tool permissions
- Agent-definition payloads
- Parallel versus sequential sub-agent spawning
- Context isolation and explicit context injection
- Sub-agent prompting patterns
- Multi-agent architecture diagnostic scenarios

## Why Use Multiple Agents?

A single agent can often complete simple tasks, but complex workflows create architectural problems.

### Context Ceiling

A single agent may need to remember many tool calls, intermediate results, constraints, and decisions. As the conversation grows, the context window can fill before the task is complete.

Tool results can accumulate rapidly, causing the agent to spend more context processing history instead of solving the remaining work.

### Sequential Bottlenecks

A single agent commonly calls tools one after another. Independent tasks that could run at the same time become queued behind each other.

If three independent tasks take times $T_1$, $T_2$, and $T_3$:

```text
Sequential time = T1 + T2 + T3
Parallel time   = max(T1, T2, T3)
```

Sequential execution increases latency and can make the system unnecessarily slow.

### Specialization Gap

One general-purpose agent is not necessarily excellent at research, writing, analysis, validation, and synthesis at the same time. A specialized agent can have a narrower context, clearer instructions, and tools suited to one responsibility.

## Multi-Agent Blueprint

Multi-agent architecture divides a complex request into smaller, scoped responsibilities. A coordinator delegates work to specialized agents and combines their results.

```text
                         Coordinator
                      /       |       \
                     /        |        \
          Research Agent  Writer Agent  Review Agent
                     \        |        /
                      \       |       /
                         Final synthesis
```

The coordinator is responsible for orchestration. Sub-agents perform focused work and report their results to the coordinator.

## Hub-and-Spoke Topology

The hub-and-spoke pattern has one central coordinator, or hub, and multiple independent sub-agents, or spokes.

### Routing Rule

All communication routes through the coordinator.

- Sub-agents communicate with the coordinator.
- Sub-agents do not directly communicate with one another.
- Each sub-agent has an isolated responsibility and context.
- The coordinator collects and synthesizes the results.

This topology avoids hidden dependencies between workers and gives the coordinator visibility into the overall workflow.

### Advantages

- Independent work can run in parallel.
- Each sub-agent has a focused purpose.
- Tool access can be scoped per agent.
- The coordinator can handle failures and retries.
- The final output has one clear owner.
- The architecture is easier to observe and debug.

Sub-agents should generally return their work to the coordinator rather than producing the final user-facing answer independently.

## Coordinator Responsibilities

### 1. Task Decomposition

Break the larger request into scoped, assignable pieces.

The coordinator decides:

- What work needs to be done
- Which tasks are independent
- Which tasks depend on earlier results
- Whether a sub-agent is necessary
- How many sub-agents should be created

### 2. Delegation

Choose the appropriate specialist for each task. For example:

- A research agent finds and evaluates sources.
- A writing agent drafts an article.
- A data agent analyzes structured information.
- A review agent checks quality and consistency.

### 3. Result Aggregation

Collect independent outputs and synthesize them into one coherent result. The coordinator must preserve the relationship between each task and its result.

Poor aggregation can produce incomplete, duplicated, or disconnected output even when the individual sub-agents performed correctly.

### 4. Error Handling

Sub-agents can fail, return incomplete information, or report that they need more input. The coordinator should handle these failures deliberately by retrying, reassigning, asking for clarification, or reporting the limitation.

## The `task` Tool

In Claude's multi-agent configuration, the coordinator uses the `task` tool to spawn sub-agents.

The coordinator must have access to the `task` tool. Without that permission, it cannot delegate work or create sub-agents.

Conceptually:

```text
Coordinator -> task tool -> New specialized sub-agent
```

The `task` entry must be explicitly included in the coordinator's allowed tools. Merely defining other tools does not give the coordinator permission to spawn agents.

## Agent Definition Payload

When the coordinator calls the `task` tool, it must provide a strict agent definition. The core fields are:

1. `description`
2. `prompt`
3. `allowed_tools`
4. `model`

### `description`

Defines the sub-agent's purpose and specialization. It helps identify what the sub-agent is designed to do.

Example:

```text
Web research specialist who finds and evaluates recent sources.
```

### `prompt`

Defines the specific goal, constraints, context, and success criteria for the task.

Example:

```text
Find five to eight recent studies about offshore wind energy and summarize the evidence relevant to the research question.
```

### `allowed_tools`

Lists the tools that this individual sub-agent is permitted to access. Tool access should be limited to what the worker needs.

### `model`

Specifies the model that should run the sub-agent.

Example shape:

```json
{
  "description": "Web research specialist",
  "prompt": "Find and summarize recent studies about offshore wind energy.",
  "allowed_tools": ["web_search", "read_page"],
  "model": "claude-model"
}
```

The exact model identifier and SDK schema should be checked against the current Anthropic documentation. For exam questions, remember the four conceptual fields and the purpose of each one.

## Task-Tool Permission Matrix

### Coordinator

The coordinator needs `task` access so it can spawn sub-agents.

### Standard Worker

A standard worker should not receive `task` access. It performs its assigned work and reports to the coordinator.

Giving every worker permission to spawn more workers can create uncontrolled recursion, unclear ownership, and unnecessary cost.

### Hierarchical Sub-Coordinator

A sub-coordinator may receive `task` access when the architecture explicitly requires a deeper delegation layer.

```text
Root coordinator
    -> Research sub-coordinator
        -> Source specialist
        -> Evidence specialist
```

This is a deliberate hierarchical design, not an accidental permission granted to every worker.

| Agent type | `task` access | Reason |
| --- | --- | --- |
| Root coordinator | Required | Spawns and manages sub-agents |
| Standard worker | Usually prohibited | Performs one scoped task |
| Explicit sub-coordinator | Allowed when designed | Delegates to a deeper layer |

## Parallel Sub-Agent Spawning

The coordinator can spawn independent sub-agents in parallel.

### Exam Rule

To spawn multiple sub-agents in parallel, emit multiple `task` tool calls in a single coordinator response.

```text
One coordinator response:
    task call 1 -> Research agent
    task call 2 -> News agent
    task call 3 -> Government-report agent
```

These are multiple tool calls in one response, not separate coordinator turns.

### Parallel Versus Sequential Timing

For independent tasks:

```text
Parallel execution   = max(T1, T2, T3)
Sequential execution = T1 + T2 + T3
```

If a system expected to run in parallel takes roughly three times longer, inspect whether the coordinator is making one task call per turn instead of emitting multiple task calls together.

### When Sequential Work Is Correct

Sequential execution is appropriate when a later task depends on the output of an earlier task. For example, a writer may need the researcher's findings before drafting an article.

Do not parallelize tasks that have an actual dependency. Parallelize independent tasks to reduce latency.

## Context Isolation

Sub-agents start with a blank context. They do not automatically inherit:

- The coordinator's conversation history
- The coordinator's system instructions
- Another sub-agent's conversation
- The results of other tasks
- Project assumptions known only to the coordinator

This isolation is intentional. It limits context growth and keeps each worker focused, but it also means the coordinator must provide any information the worker needs.

## Explicit Context Injection

The coordinator must inject relevant context into the sub-agent's `prompt` when creating the task.

```text
Coordinator context
    -> Extract relevant facts
    -> Include facts in the task prompt
    -> Spawn isolated sub-agent
```

The prompt may include:

- The specific research question
- Relevant user requirements
- Constraints and assumptions
- Results from an earlier dependent task
- Required output format
- Evaluation criteria

If a sub-agent produces irrelevant, duplicated, or context-unaware output, a likely root cause is missing explicit context injection.

### Context Injection Example

Weak task prompt:

```text
Research offshore wind energy.
```

Stronger task prompt:

```text
Research offshore wind energy for a report aimed at technical decision-makers.
Focus on studies published in the last three years, compare the main findings,
identify disagreements, and return a source-linked summary in JSON.
```

The stronger prompt supplies goal, audience, scope, constraints, and output expectations without assuming that the sub-agent knows the coordinator's history.

## Good and Bad Sub-Agent Prompting

### Good Prompting

Specify:

- The goal
- The relevant context
- Constraints and guardrails
- The expected output
- The criteria for a successful result

Allow the sub-agent to choose an appropriate method within those boundaries.

### Bad Prompting: Overly Rigid Steps

Avoid prescribing a fragile sequence such as:

```text
Open exactly this website, click these links, copy these fields, and then write the report.
```

This can fail when the site changes, becomes unavailable, or does not contain the expected information. Give the sub-agent a clear goal and constraints, rather than unnecessary implementation steps.

### Important Distinction

Good prompting is not the same as vague prompting. The coordinator should provide enough context and guardrails for reliable work while allowing the specialist to adapt its method.

## Result Synthesis

The coordinator must produce a unified output from the worker results.

Example research workflow:

```text
Research papers agent
News articles agent
Government reports agent
Social sentiment agent
              |
              v
      Coordinator synthesis
              |
              v
       Complete final answer
```

If the final output is fragmented, duplicated, or missing important connections, inspect the decomposition and aggregation responsibilities. The issue may be an overly narrow task definition or insufficient synthesis instructions.

## Exam Scenarios

### Scenario 1: Coordinator Cannot Spawn Workers

**Question:** The coordinator has research tools but cannot create sub-agents. What should be checked?

**Answer:** Confirm that `task` is explicitly included in the coordinator's allowed tools.

### Scenario 2: Every Worker Spawns More Workers

**Question:** The system keeps creating deeper agents without a clear stopping point. What is the likely design problem?

**Answer:** Standard workers have been given unnecessary `task` access. Restrict task access to the root coordinator or explicitly designed sub-coordinators.

### Scenario 3: Workers Produce Irrelevant Results

**Question:** Sub-agents return generic or context-unaware answers even though the coordinator knows the full user request. Why?

**Answer:** Sub-agents start with blank context. The coordinator must explicitly inject the required information into each agent-definition prompt.

### Scenario 4: Three Independent Tasks Are Slow

**Question:** Three independent workers run one after another, causing high latency. What should change?

**Answer:** Emit three `task` tool calls in a single coordinator response so the independent tasks can run in parallel.

### Scenario 5: A Worker Needs Previous Results

**Question:** A writing agent must use findings from a research agent. Should the writing agent automatically see the research conversation?

**Answer:** No. The coordinator must explicitly pass the relevant research results in the writing agent's prompt or task input.

### Scenario 6: Final Output Is Fragmented

**Question:** Each sub-agent returns a good result, but the final answer is duplicated and inconsistent. What responsibility was not handled correctly?

**Answer:** The coordinator's result aggregation and synthesis step needs improvement.

## Exam Checklist

- Know why context ceilings, sequential bottlenecks, and specialization gaps motivate multi-agent systems.
- Recognize the hub-and-spoke topology.
- Route sub-agent communication through the coordinator.
- Know that the coordinator decomposes, delegates, aggregates, and handles errors.
- Remember that the coordinator needs `task` access to spawn sub-agents.
- Do not give standard workers `task` access unless they are explicit sub-coordinators.
- Memorize the four agent-definition fields: `description`, `prompt`, `allowed_tools`, and `model`.
- Spawn independent workers with multiple task calls in one coordinator response.
- Understand that sub-agents start with blank context.
- Inject required context explicitly through the task prompt.
- Give sub-agents goals and guardrails rather than fragile step-by-step instructions.
- Let the coordinator synthesize worker results into the final answer.
- Distinguish legitimate sequential dependencies from accidental sequential execution.

## Final Summary

Multi-agent systems divide complex work among specialized agents while a coordinator owns orchestration and final synthesis. The coordinator uses the `task` tool to create workers, supplies each worker with a clear agent definition, runs independent tasks in parallel, injects context explicitly, handles failures, and combines the results.

For the exam, remember the central rules: **the coordinator delegates, standard workers stay focused, parallel work uses multiple task calls in one response, and every sub-agent starts with a blank context.**