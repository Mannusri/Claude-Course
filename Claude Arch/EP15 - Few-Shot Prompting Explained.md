# EP15: Few-Shot Prompting Explained

## Lesson Goal

This lesson explains how to use a small number of high-quality demonstrations to make Claude's outputs more consistent and aligned with project-specific decisions.

The main exam topics are:

- Instructions versus demonstrations
- Input, reasoning, and output structure
- The two-to-four example rule
- Boundary-case and drift targeting
- Few-shot examples for severity classification
- Few-shot examples for structured extraction
- Few-shot examples for resolve-versus-escalate decisions
- Tool descriptions before few-shot examples
- Context bloat and pattern-matching risks

## Why Few-Shot Prompting Helps

Instructions describe what the model should do, but they may leave ambiguity about how the result should look.

For example, a prompt may say:

```text
Classify code-review findings as critical, high, or medium.
```

The model understands the task but may classify the same kind of issue differently across runs because the boundaries between the labels are not concrete enough.

Few-shot prompting adds demonstrations:

```text
Input -> Reasoning -> Expected output
```

The demonstrations show the model how the team applies the rule to real cases.

## Prose Versus Demonstration

Long prose can describe a policy, but an example demonstrates the policy in operation.

```text
Detailed instruction: Explain what a critical issue means.

Demonstration: Show a code pattern, explain why it is critical,
and return the exact critical label and output shape.
```

The exam principle is that targeted few-shot examples are often more effective than adding more descriptive prose when the problem is inconsistent classification or output behavior.

## The Anatomy of a Good Example

A high-quality example has three parts:

1. **Input**: The case or content the model receives.
2. **Reasoning**: The project-specific rationale for the decision.
3. **Output**: The exact label, action, or structured result expected.

### Example Structure

```text
Input:
The authentication check exists but is bypassed, so every request proceeds
without validating the token.

Reasoning:
This is an exploitable authentication failure with direct security impact.
It belongs to the critical category.

Output:
{
  "severity": "critical",
  "category": "authentication_bypass"
}
```

### Why Reasoning Matters

An output-only example encourages surface pattern matching. A reasoning-backed example teaches the decision rule that produced the output, helping the model generalize to a new case that is not identical to the demonstration.

## The Two-to-Four Example Rule

Use approximately two to four targeted examples for a decision boundary or output behavior.

This is a practical sweet spot:

- Enough variety to demonstrate the rule
- Small enough to preserve context
- Focused enough to avoid confusing the model
- Efficient enough for repeated production calls

### Why Not Six to Eight?

Too many examples can:

- Bloat the context
- Increase token cost
- Dilute the important rule
- Encourage literal pattern matching
- Create conflicting or redundant demonstrations

The goal is not to provide every possible case. The goal is to select a few examples that expose the rule and its boundaries.

## Choosing Examples

Prefer examples that cover:

- Different input structures
- Ambiguous boundary cases
- Common failure modes
- Cases where the model previously drifted
- Both positive and negative decisions
- Different output formats that should map to the same schema

Do not spend all examples on obvious happy paths that the model already handles reliably.

## Diagnostic Workflow for Example Selection

Use observed model behavior to select demonstrations:

1. Run representative inputs multiple times.
2. Identify where the output changes or drifts.
3. Find the decision boundary causing the inconsistency.
4. Create examples that explain that boundary.
5. Include the reasoning and expected output.
6. Re-run the same inputs and new nearby cases.
7. Keep only the examples that improve reliability without bloating the prompt.

This is more effective than adding random examples.

## Few-Shot Severity Classification

A code-review agent may need to classify findings as `critical`, `high`, or `medium`.

### Critical Example

```text
Input:
User input is concatenated directly into a SQL query.

Reasoning:
An attacker can alter the query and access or modify data. This is a
credible, directly exploitable security vulnerability.

Output:
{
  "severity": "critical",
  "category": "sql_injection"
}
```

### High Example

```text
Input:
A payment operation can fail after charging the customer, but the failure
is not logged or reconciled.

Reasoning:
This creates a significant correctness issue on a critical path, but it
requires the payment failure scenario to occur.

Output:
{
  "severity": "high",
  "category": "missing_payment_error_handling"
}
```

### Medium Example

```text
Input:
A non-critical administrative list endpoint performs an avoidable small
amount of repeated work.

Reasoning:
The issue is real but has limited scope and no immediate security or data
integrity impact.

Output:
{
  "severity": "medium",
  "category": "limited_efficiency_issue"
}
```

The examples define how the project maps reasoning to labels. They are more useful than simply saying “critical is very serious.”

## Few-Shot Structured Extraction

Few-shot examples can show an extraction agent how to handle different document shapes.

### Example 1: Structured Table

```text
Input:
| Field | Value |
| Contract value | $45,000 |
| Effective date | 2026-01-01 |
| Payment terms | Net 30 |

Reasoning:
The requested values are explicitly present in a table. No inference is needed.

Output:
{
  "contract_value": "$45,000",
  "effective_date": "2026-01-01",
  "payment_terms": "Net 30"
}
```

### Example 2: Prose Document

```text
Input:
The agreement begins on January 1, 2026, for a total value of $45,000.
Payment must be made within thirty days of invoice.

Reasoning:
The same fields are explicitly stated in prose rather than a table. Extract
the values and normalize only according to the stated output schema.

Output:
{
  "contract_value": "$45,000",
  "effective_date": "2026-01-01",
  "payment_terms": "Net 30"
}
```

These examples teach the model that different source layouts can produce the same structured output.

### Extraction Safety

Few-shot examples must reinforce non-fabrication rules:

- Extract only information supported by the source.
- Return `null` when a field is absent.
- Preserve raw text when a value is ambiguous.
- Never infer a medical, financial, or identity value that is not stated.

## Few-Shot Resolve Versus Escalate

A customer-support agent may need to decide whether to resolve a case directly or escalate it.

### Resolve Example

```text
Input:
My invoice shows a small extra charge. I have been a customer for four years.

Reasoning:
This is a limited billing discrepancy with no service failure or repeated
failed resolution. The trusted customer context supports a small goodwill
resolution.

Output:
{
  "decision": "resolve",
  "action": "apply approved credit and close the ticket"
}
```

### Escalate Example

```text
Input:
I was charged three times for the same order and support has failed to fix it
for two weeks.

Reasoning:
This is a repeated billing failure with unsuccessful prior support contact.
It exceeds a simple discrepancy and requires escalation.

Output:
{
  "decision": "escalate",
  "routing_tier": "senior_support"
}
```

Use examples that straddle the ambiguous boundary between self-resolution and escalation. Obvious examples provide less value.

## Tool Descriptions Before Few-Shot Examples

Few-shot examples improve the model's decision behavior after it has the relevant context. In an agent with tools, the model must first know which tool to select.

Recommended order:

1. Define clear, mutually exclusive tool names and descriptions.
2. Scope the tools to the agent's role.
3. Add few-shot examples for classification or action boundaries.
4. Validate the resulting tool calls and output.

If tool descriptions are ambiguous, few-shot examples may not solve the routing problem. Fix tool routing first.

```text
Clear tool descriptions -> Correct tool selected -> Few-shot reasoning guides the decision
```

## Few-Shot Examples Versus Other Techniques

| Problem | Best first technique |
| --- | --- |
| Tool is not selected or wrong tool is selected | Improve tool name and description |
| Critical policy must never be violated | Programmatic hook or application gate |
| Same type of issue receives inconsistent labels | Two to four reasoning-backed examples |
| Output cannot be consumed by a pipeline | Structured schema and validation |
| Missing source data is fabricated | Explicit null and non-inference constraints |
| Large exploration fills context | Explore sub-agent or isolated session |

Few-shot prompting is not a replacement for deterministic safety controls, tool definitions, or schemas.

## Avoiding Example Overfitting

Examples should teach the underlying decision rule, not force the model to memorize exact strings.

Avoid:

- Repeating nearly identical examples
- Giving only one narrow wording
- Omitting the rationale
- Covering only easy cases
- Including contradictory outputs
- Providing a large catalog of examples instead of a clear rule

Use varied wording and structure while keeping the reasoning consistent.

## Example Maintenance

Few-shot examples are part of the prompt contract and should evolve with the project.

Review them when:

- Severity definitions change
- A new failure mode appears
- Code or document formats change
- Production drift is observed
- A category is repeatedly misclassified
- The output schema changes

Keep examples representative, current, and tied to the team's actual policy.

## Exam Scenarios

### Scenario 1: Inconsistent Severity

**Question:** A CI reviewer correctly finds the same class of bug but alternates between `critical` and `high`. What is the most effective fix?

**Answer:** Add two to four targeted few-shot examples containing the input, reasoning, and expected severity output.

### Scenario 2: Too Many Examples

**Question:** A team plans to add six to eight examples for every label. What is the likely problem?

**Answer:** The prompt may become bloated and encourage pattern matching. Use a small set of varied, targeted examples instead.

### Scenario 3: Output-Only Examples

**Question:** Why should each demonstration include reasoning rather than only an input and label?

**Answer:** Reasoning teaches the underlying decision rule and helps the model generalize to novel inputs instead of matching surface patterns.

### Scenario 4: Document Shape Variation

**Question:** An extraction agent receives both tables and prose documents. How can few-shot prompting help?

**Answer:** Provide reasoning-backed examples for each relevant document structure that map to the same output schema.

### Scenario 5: Boundary Escalation

**Question:** A support agent inconsistently resolves borderline cases instead of escalating them. What examples are most useful?

**Answer:** Two or more targeted examples that straddle the ambiguous resolve-versus-escalate boundary, with explicit reasoning and action.

### Scenario 6: Wrong Tool First

**Question:** The agent selects the wrong tool even after few-shot examples were added. What should be checked first?

**Answer:** Improve the tool name, description, scope, and invocation boundaries before adding more examples.

### Scenario 7: Missing Medical Field

**Question:** The source document does not contain a patient ID. What should the example demonstrate?

**Answer:** Return `null` and never infer or fabricate the identifier.

## Exam Anti-Patterns

### 1. Replacing Criteria with Examples

Few-shot examples complement explicit criteria. They do not replace clear include and skip rules.

### 2. Output-Only Demonstrations

A label without reasoning teaches shallow pattern matching and does not clearly define the decision boundary.

### 3. Six to Eight Examples by Default

More examples are not automatically better. Excess examples can consume context and create pattern-matching behavior.

### 4. Obvious Examples Only

The most useful examples target ambiguous and drifting cases, not only cases the model already handles.

### 5. Using Examples to Enforce Safety

For non-negotiable policies, use hooks and application validation. Examples are probabilistic guidance.

### 6. Adding Examples Before Fixing Tool Routing

If the wrong tool is selected, improve tool descriptions and scope first.

### 7. Fabricating Missing Extraction Values

Examples must reinforce null handling and preservation of ambiguous source text.

## Exam Checklist

- Few-shot prompting demonstrates the desired behavior with examples.
- A strong example contains input, reasoning, and output.
- Use approximately two to four targeted examples.
- Target ambiguous boundaries, drift, and failure modes.
- Include varied structures and wording.
- Use reasoning to teach the rule behind the label.
- Use examples for severity, extraction, and resolve-versus-escalate decisions.
- Fix tool descriptions before using examples to improve agent behavior.
- Do not use few-shot examples as a replacement for hooks or schemas.
- Avoid context bloat and repetitive examples.
- Return null for missing extracted data.
- Revisit examples as project policies and input formats evolve.

## Final Summary

Few-shot prompting improves consistency by showing Claude how a team applies a rule, not merely describing the rule in abstract language. The strongest demonstrations contain an input, the reasoning that leads to a decision, and the exact expected output.

For the exam, remember: **use two to four targeted reasoning-backed examples, focus on ambiguous boundaries and observed drift, fix tool descriptions before adding examples, and never use examples instead of deterministic safety controls.**
