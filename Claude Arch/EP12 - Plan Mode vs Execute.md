# EP12: Plan Mode Versus Direct Execution

## Lesson Goal

This lesson explains when Claude should investigate and propose a plan before changing files, and when it should execute a clearly defined task directly.

The main exam topics are:

- Plan Mode versus direct execution
- Scope, ambiguity, and reversibility
- Large architectural changes
- Human review and approval before execution
- Explore sub-agents and context-window exhaustion
- Iterative refinement
- Test-driven iteration
- Concrete examples
- The interview pattern for clarifying requirements
- Model and effort selection

## The Core Decision

Not every task deserves the same workflow.

```text
Clear, local, reversible task -> Direct execution
Ambiguous, broad, architectural task -> Plan Mode first
```

Plan Mode and direct execution are not competing philosophies. They are stages that can be used together:

```text
Plan Mode -> Review and approve plan -> Direct execution
```

Use planning when the cost of a wrong approach is high. Use direct execution when the destination is already clear and corrections are cheap.

## Direct Execution

Direct execution is the default workflow for a well-defined task. Claude analyzes the request and makes the requested changes without a separate planning phase.

### Use Direct Execution When

- The scope is clear.
- The target files are known.
- The desired output is predictable.
- The change is local or small.
- There is one obvious reasonable approach.
- A mistake is cheap to correct.
- The change is easy to review and revert.

### Examples

- Fix a clear bug from a stack trace in one file.
- Add unit tests for an existing function.
- Update a known configuration value.
- Generate boilerplate from a clear specification.
- Change a function signature in an identified file.
- Apply a small, well-understood UI adjustment.

### Direct-Execution Example

```text
In src/agent.py, change process_refund to accept a list of order IDs.
Update its unit tests and run the targeted test file.
```

The destination, scope, and validation path are clear.

## Plan Mode

Plan Mode is an investigation and design phase before code is changed. Claude reads broadly, understands the current architecture, identifies dependencies, evaluates alternatives, and presents a proposed course of action.

A typical Plan Mode workflow is:

1. Inspect the relevant codebase.
2. Map dependencies and constraints.
3. Identify risks and affected components.
4. Compare valid implementation approaches.
5. Describe the recommended sequence.
6. Ask the user to review or approve the plan.
7. Switch to execution after approval.

### Use Plan Mode When

- The change affects many files.
- The architecture may change.
- Requirements are ambiguous.
- Multiple valid approaches exist.
- The change is expensive to reverse.
- Dependencies are tightly coupled or unclear.
- A migration, refactor, or re-architecture is required.
- The initial assumptions may be wrong.
- A human approval checkpoint is important.

### Examples

- Refactor a monolith into services.
- Migrate a tightly coupled system to a new data-access layer.
- Re-architect authentication.
- Change a framework or persistence strategy.
- Extract database queries from a large codebase into an ORM.
- Create a new application foundation from requirements.

## Decision Signals

Look for these exam signals:

| Signal in the scenario | Recommended approach |
| --- | --- |
| Clear stack trace and one-file fix | Direct execution |
| Known file and exact requested change | Direct execution |
| Large-scale refactor | Plan Mode |
| Multiple valid architectures | Plan Mode |
| Migration or re-architecture | Plan Mode |
| Hard or expensive to reverse | Plan Mode |
| Tightly coupled dependencies | Plan Mode |
| Ambiguous open-ended requirements | Plan Mode plus interview questions |
| New application foundation | Plan Mode |
| Context window filling during discovery | Explore sub-agent |

### Fast Decision Rule

Ask four questions:

1. Does the change affect more than one file non-trivially?
2. Are there multiple reasonable approaches?
3. Would reversing a wrong approach be expensive?
4. Could the change alter the architecture or system boundaries?

If any answer is strongly yes, start with Plan Mode.

## Why Direct Execution Can Fail for Large Changes

Suppose Claude is asked to convert a large monolith into microservices without a plan. It may:

- Read one file at a time.
- Follow dependencies as it discovers them.
- Modify files before the full architecture is understood.
- Create inconsistent intermediate states.
- Break imports and tests.
- Choose an approach that is difficult to reverse.
- Require a costly repair or rollback.

The problem is not that Claude cannot edit files. The problem is that execution began before the system-level design was understood.

## Plan Mode as a Human Checkpoint

Plan Mode makes important decisions visible before side effects occur. The user can review:

- Proposed folder structure
- Affected modules
- Data models
- API boundaries
- Migration sequence
- Testing strategy
- Risks and tradeoffs
- Alternative designs

The user can reject or revise the plan before any large change is made.

This is especially important for production systems, destructive operations, security-sensitive changes, and work that changes architectural boundaries.

## Planning a New Application

Plan Mode is useful even when starting from an empty repository. A new project has no legacy constraints, but early architectural choices still affect:

- Folder structure
- Module boundaries
- Data models
- API contracts
- Testing strategy
- Authentication design
- Deployment shape
- Future extensibility

A good planning request might say:

```text
Before writing code, propose the folder structure, core modules,
data models, API endpoints, testing strategy, and major tradeoffs.
Do not create files until I approve the plan.
```

## Plan Mode and Direct Execution Together

The recommended sequence for a large change is:

```text
1. Plan
2. Review
3. Approve or revise
4. Execute
5. Test
6. Refine
```

Do not remain in Plan Mode forever. The purpose is to make the execution safer and more deliberate, not to avoid implementation.

## Explore Sub-Agent

Large codebases can exhaust the main session's context during discovery. Claude may need to inspect many files, follow imports, trace calls, and read tests before it can make a plan.

Use an explore sub-agent to perform broad discovery separately:

```text
Main session
    -> Explore sub-agent reads and maps the codebase
    -> Compact summary returns
    -> Main session creates or reviews the plan
```

### What the Explore Sub-Agent Does

- Reads broadly across the codebase.
- Maps architecture and dependencies.
- Finds relevant entry points.
- Follows import and call chains.
- Reviews related tests.
- Returns a compact summary to the main session.

### Exam Fact

When the scenario mentions context-window exhaustion during large codebase discovery, choose an explore sub-agent. It keeps broad exploratory work out of the main session and returns a concise summary.

### Explore Sub-Agent Versus Plan Mode

These solve different problems:

| Mechanism | Primary purpose |
| --- | --- |
| Explore sub-agent | Reduce main-session context pressure during discovery |
| Plan Mode | Design and review the implementation approach |

They can be combined:

```text
Explore sub-agent -> Plan Mode -> User approval -> Direct execution
```

## Iterative Refinement

The first generated result is not always the best result. Iterative refinement uses concrete feedback, tests, and examples to improve the output.

```text
Generate -> Review -> Give specific feedback -> Revise -> Validate
```

### Specific Feedback

Avoid vague feedback such as:

```text
Make it better.
```

Use actionable feedback:

```text
The error handler in process_refund does not log the full exception object.
Preserve the original error, add the order ID to the log context, and update
 the test that checks the failure path.
```

Specific feedback gives Claude a clear change, location, and success condition.

## Test-Driven Iteration

In test-driven iteration:

1. Provide or define the expected tests.
2. Ask Claude to implement the change.
3. Run the tests.
4. Feed failures and relevant logs back to Claude.
5. Ask Claude to revise the implementation.
6. Repeat until the tests pass.

Machine-generated test failures are precise feedback. They reduce ambiguity about what success means.

### Exam Signal

If a question asks how to improve implementation reliability through repeated validation, choose test-driven iteration and feed concrete failures back into the loop.

## Concrete Examples

Concrete examples often communicate requirements more reliably than long prose.

For a transformation task, provide:

- Before input
- Expected after output
- Edge cases
- Invalid examples
- Required formatting

Example:

```json
{
  "input": {
    "status": 3
  },
  "expected_output": {
    "status": "shipped"
  }
}
```

A few precise examples can clarify structure, naming, and edge behavior. They improve consistency, but they do not replace deterministic application-level enforcement for critical policies.

## The Interview Pattern

The interview pattern asks Claude to identify missing requirements before implementation.

Example:

```text
Build a REST API for user management.
Before writing code, ask me any clarifying questions about requirements,
constraints, security, integrations, and preferences that could change the design.
```

Claude can ask about:

- Authentication and authorization
- Data model and storage
- API versioning
- Validation requirements
- Error format
- Performance expectations
- Deployment environment
- Backward compatibility
- Testing requirements

### When to Use the Interview Pattern

Use it when:

- Requirements are ambiguous.
- Several valid designs exist.
- Important constraints are unknown.
- Rework would be expensive.
- First-pass quality matters.
- The user may know the domain constraints but has not stated them yet.

### Interview Pattern Versus Plan Mode

The interview pattern is a prompting technique. Plan Mode is an execution mode or workflow stage. They can be used together:

```text
Interview questions -> Plan Mode investigation -> User approval -> Execution
```

## Model and Effort Selection

Model selection should match the task's complexity and the value of additional reasoning.

### Lower-Cost Model or Moderate Effort

Often suitable for:

- Straightforward scaffolding
- Boilerplate generation
- Small known edits
- Simple test creation
- Predictable transformations

### Stronger Model or Higher Effort

May be justified for:

- Difficult debugging
- Complex architectural planning
- Cross-module reasoning
- Ambiguous requirements
- Repeated failed attempts
- High-risk migrations

A practical workflow can start with a capable lower-cost model for predictable work and switch to a stronger model or higher effort when the task is stuck or reasoning demands increase.

The goal is not to use the most expensive model for every task. Match model effort to uncertainty, complexity, and risk.

## Example Decision Walkthroughs

### Small Bug with a Clear Stack Trace

```text
Input: A stack trace points to one function in one file.
Scope: One file.
Risk: Low and easy to revert.
Approach: Direct execution, then run the targeted test.
```

### Monolith to Microservices

```text
Input: A tightly coupled production monolith must become services.
Scope: Many modules and boundaries.
Risk: High and expensive to reverse.
Approach: Plan Mode, dependency exploration, alternatives, approval, execution.
```

### Large Codebase Investigation

```text
Input: Understand a 200-file system before changing it.
Risk: Main-session context exhaustion.
Approach: Use an explore sub-agent, return a compact summary, then plan.
```

### Ambiguous API Request

```text
Input: Build a user-management API with few requirements.
Risk: Many valid designs and likely rework.
Approach: Use the interview pattern to ask clarifying questions, then plan.
```

## Exam Scenarios

### Scenario 1: Large Refactor

**Question:** A tightly coupled Node.js monolith must be migrated to a microservice architecture. What should happen first?

**Answer:** Use Plan Mode to inspect dependencies, compare a phased strategy, and obtain approval before making broad changes.

### Scenario 2: Clear Single-File Fix

**Question:** A stack trace identifies one function in one file, and the required fix is precise. Which mode fits?

**Answer:** Direct execution, followed by targeted tests.

### Scenario 3: Multiple Valid Approaches

**Question:** A change could be implemented through several valid architectural approaches, and reversing the wrong choice would be expensive. What should happen?

**Answer:** Use Plan Mode and present the tradeoffs before execution.

### Scenario 4: Context Exhaustion

**Question:** Claude must understand a very large codebase, but reading the dependencies fills the main session before planning begins. What should be used?

**Answer:** An explore sub-agent that performs discovery and returns a compact summary.

### Scenario 5: Ambiguous Requirements

**Question:** You ask Claude to build a REST API but have not specified authentication, persistence, or compatibility requirements. What improves first-pass quality?

**Answer:** Use the interview pattern and ask Claude to gather clarifying requirements before implementation.

### Scenario 6: Failing Tests

**Question:** Claude generated an implementation, but three assertions fail. What is the best next step?

**Answer:** Feed the specific test failures and relevant logs back to Claude and iterate on the implementation.

### Scenario 7: New Application Foundation

**Question:** You are starting a new application and want to evaluate folder structure, modules, models, and API boundaries before files are created. Which mode fits?

**Answer:** Plan Mode.

## Exam Anti-Patterns

### 1. Directly Executing a Large Migration

Do not begin a high-risk architectural migration by editing files immediately. Inspect, plan, compare, and approve first.

### 2. Using Plan Mode for Every Tiny Change

Plan Mode has overhead. A clear, local, reversible change should use direct execution.

### 3. Guessing Dependencies

Do not guess the dependency graph in a tightly coupled codebase. Use exploration and Plan Mode.

### 4. Reading Everything in the Main Session

Broad discovery can exhaust context. Use an explore sub-agent for large codebases.

### 5. Giving Vague Feedback

“Make it better” does not define a fix. Point to the behavior, location, and expected result.

### 6. Stopping After the First Output

Use tests, concrete examples, and specific feedback to refine the result.

### 7. Confusing Interview Pattern with Plan Mode

The interview pattern gathers requirements through prompting. Plan Mode is the structured investigation and approval stage. They can be combined.

### 8. Using the Most Expensive Model Automatically

Use model strength and effort proportional to task complexity, uncertainty, and risk.

## Exam Checklist

- Use direct execution for clear, local, predictable, reversible changes.
- Use Plan Mode for broad, ambiguous, architectural, or expensive-to-reverse changes.
- Plan Mode investigates before modifying files and provides an approval checkpoint.
- Plan Mode and direct execution are sequentially compatible.
- Use an explore sub-agent to prevent main-session context exhaustion during broad discovery.
- Use the interview pattern to gather missing requirements before implementation.
- Use specific feedback for iterative refinement.
- Use tests and failure logs as machine-generated feedback.
- Use concrete before-and-after examples for transformations.
- Match model and effort to uncertainty, complexity, and risk.
- Do not guess dependencies in a large or tightly coupled system.
- Do not bulk-read a large codebase in the main session when an explore sub-agent is appropriate.

## Final Summary

Direct execution is best when the destination is known and mistakes are cheap to fix. Plan Mode is best when the task is broad, ambiguous, architectural, or expensive to reverse. The strongest workflow often combines exploration, planning, approval, execution, and iterative validation.

For the exam, remember: **clear single-file change means direct execution; large refactor means Plan Mode; context exhaustion means an explore sub-agent; ambiguous requirements mean the interview pattern; failing tests mean concrete iterative feedback.**
