# EP16: Structured Output and JSON Schema

## Lesson Goal

This lesson explains how to make agent and tool outputs reliable enough for downstream systems, sub-agents, coordinators, and production pipelines.

The main exam topics are:

- Why free-form output breaks pipelines
- Fake-tool structured output
- JSON Schema and `tool_choice`
- Required versus nullable fields
- Structural validation versus semantic validation
- Self-correcting format retry loops
- Missing data and honest null values
- Cross-field and business-rule checks
- Human or coordinator handoff after failed retries

## Why Free-Form Output Fails

A model can understand the requested data but still return it in a form that a downstream parser cannot safely consume.

Common failures include:

- Markdown code fences around JSON
- Missing fields
- Renamed keys
- Changed nesting
- Strings where numbers are required
- Inconsistent date formats
- Fabricated values for absent fields
- Correct-looking structure containing the wrong value

Example of fragile output:

```text
```json
{"vendor": "Example", "total": 45000}
```
```

A parser expecting the first character to be `{` may fail when it receives a backtick or explanatory prose.

## Tool-Based Structured Output

A robust approach is to define a tool whose purpose is to accept the required structured payload. This is sometimes called the fake-tool or structured-output-tool pattern.

The tool does not need to perform a business operation. Its purpose is to enforce a typed contract for the model's response.

```text
Document or tool results
        |
        v
Claude selects extraction tool
        |
        v
Tool input must match JSON Schema
        |
        v
Application reads the structured input
```

### Example Fake Tool

```json
{
  "name": "extract_invoice_data",
  "description": "Extract invoice fields from the supplied document.",
  "input_schema": {
    "type": "object",
    "properties": {
      "vendor_name": {
        "type": ["string", "null"]
      },
      "total_amount": {
        "type": ["number", "null"]
      },
      "invoice_date": {
        "type": ["string", "null"]
      }
    },
    "required": ["vendor_name", "total_amount", "invoice_date"]
  }
}
```

The tool's application function may simply accept and return the validated payload. The important behavior is that Claude must produce the requested shape.

## Why Tool Use Is Stronger Than Prompt-Only JSON

A prompt such as:

```text
Return the answer as JSON.
```

is guidance, not a strict contract. Claude may still add Markdown, omit fields, rename keys, or change nesting.

A tool input schema provides a stronger structural boundary:

- Field names are defined.
- Types are defined.
- Required properties are defined.
- Nullable values can be explicitly allowed.
- The application can parse a known input structure.

For strict extraction, use tool use or the platform's supported structured-output mechanism rather than relying only on prose instructions.

## `tool_choice` for Structured Output

The tool choice determines whether and which tool Claude must call.

### `auto`

```json
{
  "type": "auto"
}
```

Claude may call a tool or return ordinary text. This is useful for conversational workflows but unreliable when a structured extraction tool must run every time.

### `any`

```json
{
  "type": "any"
}
```

Claude must call at least one of the available tools, but Claude chooses which tool.

Use `any` when:

- A tool call is mandatory.
- One of several structured tool outputs is acceptable.
- The response must contain a tool-use block.

### Exact `tool`

```json
{
  "type": "tool",
  "name": "extract_invoice_data"
}
```

Claude must call the named tool.

Use exact `tool` selection when one specific extraction schema is required.

### Comparison

| Mode | Tool call required? | Who selects the tool? | Extraction use |
| --- | --- | --- | --- |
| `auto` | No | Claude, if it calls one | Optional tool use |
| `any` | Yes, at least one | Claude | One of several acceptable schemas |
| `tool` | Yes, exact tool | Application | One required schema |

### Exam Rule

If the question says that Claude must produce structured output using an available tool, choose `any`. If it must use one named extraction tool, choose exact `tool`. Do not choose `auto` for a strict extraction pipeline.

## Required Versus Nullable Fields

A required field is not always a field that must contain a non-null value.

This distinction is essential:

```text
Required + non-nullable -> Field must exist and contain a value
Required + nullable      -> Field must exist, but null is valid
```

### Unsafe Schema

```json
{
  "type": "object",
  "properties": {
    "tax_id": {
      "type": "string"
    }
  },
  "required": ["tax_id"]
}
```

If the source document does not contain a tax ID, the model may invent one to satisfy the non-nullable requirement.

### Honest Schema

```json
{
  "type": "object",
  "properties": {
    "tax_id": {
      "type": ["string", "null"]
    }
  },
  "required": ["tax_id"]
}
```

Now `tax_id` must be present, but the honest value can be `null` when the source does not contain it.

### Exam Rule

When source data may be absent, use a required nullable field instead of forcing the model to fabricate a value.

## Honest Extraction Rules

A schema should be combined with explicit extraction instructions:

```text
Extract only information explicitly present in the source.
If a field is absent, return null.
If a value is ambiguous, preserve the raw text and mark it ambiguous.
Never infer a patient ID, diagnosis, dosage, invoice date, or financial value.
```

A schema controls shape; instructions and validation control truthfulness.

## Syntax Versus Semantics

A valid schema does not guarantee that the data is correct.

### Structural or Syntax Errors

Examples:

- Malformed JSON
- Missing required property
- Wrong primitive type
- Invalid nesting
- Incorrect field name
- Invalid date format

Use:

- JSON Schema
- Tool-use enforcement
- Schema validators
- Format retry loops

### Semantic Errors

Examples:

- Reading the wrong row from a table
- Assigning an invoice date to a due-date field
- Confusing two similar customer records
- Returning a value from the wrong document section
- Fabricating a value that is structurally valid
- Violating a business rule

Use:

- Business-logic validation
- Cross-field consistency checks
- Source attribution checks
- Domain-specific validators
- Human review or escalation
- Post-tool hooks where appropriate

### Boundary Matrix

| Failure | Example | Appropriate fix |
| --- | --- | --- |
| Structural | Missing `invoice_date` key | Schema validation or retry |
| Structural | Number returned as a string | Schema validation or retry |
| Structural | Malformed JSON | Schema validation or retry |
| Semantic | Wrong table row selected | Business or cross-field validation |
| Semantic | Fabricated missing value | Nullable field and non-fabrication rule |
| Semantic | Incorrect customer-order association | Business-rule validation |
| Semantic | Correct shape, false amount | Source and domain validation |

### Critical Exam Rule

A schema can prove that the output has the right shape. It cannot prove that the values are truthful or correctly interpreted.

## Self-Correcting Format Retry Loop

A retry loop can correct structural failures by returning validation feedback to Claude.

```text
1. Request structured tool output.
2. Validate the result.
3. If the format is invalid, create specific feedback.
4. Append the assistant response and validation feedback correctly.
5. Ask Claude to produce the corrected structure.
6. Retry only up to a bounded maximum.
7. Return success or escalate after the limit.
```

Conceptual flow:

```text
Extraction
    -> Schema validation
    -> valid? ---- yes ----> Return structured result
    -> no
    v
Specific format error
    -> Claude correction
    -> bounded retry
```

### Good Feedback

```text
Validation failed: `invoice_date` must match YYYY-MM-DD.
The field was present but used the format 01/02/2026.
Return the same data with only the date normalized to YYYY-MM-DD.
```

### Weak Feedback

```text
Try again.
```

Specific feedback helps Claude correct the structural defect without changing unrelated values.

## Retry Limits

Retries should be bounded:

```python
max_retries = 2
```

If the output remains structurally invalid after the limit:

- Return a structured failure.
- Escalate to the coordinator or human reviewer.
- Preserve the validation errors.
- Do not retry indefinitely.

## What Retries Cannot Do

A format retry loop is a format corrector, not a data generator.

If a source document genuinely lacks a field, retries cannot discover the missing fact. Repeatedly telling the model to “look harder” may encourage hallucination.

Correct approach:

```text
Missing source field -> null or explicit missing state
Invalid format      -> validation feedback and bounded retry
Wrong interpretation -> semantic validation or human review
```

## Semantic Validation

After schema validation, apply domain checks.

Examples:

- Invoice due date should not precede invoice date.
- Order must belong to the verified customer.
- Refund amount must not exceed the order amount.
- Extracted total should equal the sum of line items when applicable.
- Patient identifiers should match the source exactly.
- Currency and amount fields must be consistent.

These checks can be implemented in application code, pre- or post-tool hooks, or a dedicated validation agent.

## Cross-Field Consistency

A response can be individually valid by field and still be wrong as a whole.

Example:

```json
{
  "invoice_date": "2026-02-01",
  "due_date": "2026-01-01",
  "payment_terms": "Net 30"
}
```

Every field has a valid type, but the dates conflict with the stated payment terms. Cross-field validation should reject or escalate the result.

## Human and Coordinator Handoff

Escalate when:

- Maximum format retries are exhausted.
- Semantic validation fails.
- Source evidence is contradictory.
- A required business decision is unavailable.
- The result affects a high-risk domain.
- The model cannot distinguish between plausible interpretations.

The handoff should include:

- Original request
- Structured output attempted
- Validation errors
- Source references
- Retry count
- Semantic conflicts
- Required next action

## Exam Scenarios

### Scenario 1: Markdown-Wrapped JSON

**Question:** A downstream parser fails because Claude wraps valid JSON in Markdown fences. What is the strongest architectural fix?

**Answer:** Use a structured-output tool or supported schema-enforced tool use rather than relying on prompt-only JSON instructions.

### Scenario 2: Mandatory Extraction Tool

**Question:** An invoice extraction tool must be called every time. Which `tool_choice` mode fits?

**Answer:** Use exact `tool` selection with the extraction tool's name, or `any` if any one of several valid extraction tools is acceptable.

### Scenario 3: Optional Tool Call

**Question:** Claude may answer conversationally or call a tool when useful. Which mode fits?

**Answer:** `auto`.

### Scenario 4: Missing Tax ID

**Question:** Documents genuinely omit a tax ID, but the schema currently requires a non-null string. What should change?

**Answer:** Keep the field required but make its type nullable so the valid missing value is `null`.

### Scenario 5: Valid Schema, Wrong Amount

**Question:** The output passes JSON Schema but reads the wrong table row and returns an incorrect amount. Will more format retries solve it?

**Answer:** No. This is a semantic error. Add source-aware, business-rule, or cross-field validation.

### Scenario 6: Repeated Validation Failure

**Question:** A field is missing from every source document and the retry loop fails on every attempt. What should happen?

**Answer:** Represent the field as nullable and return `null`; do not force the model to generate a value.

### Scenario 7: Format Correction

**Question:** The output contains `invoice_date: 01/02/2026`, but the schema requires `YYYY-MM-DD`. What should the retry feedback say?

**Answer:** Identify the exact field and required format, then ask Claude to correct the format with a bounded retry.

## Exam Anti-Patterns

### 1. Prompt-Only JSON

Saying “return JSON” does not guarantee a stable schema or parser-safe output.

### 2. Required Non-Nullable Fields for Optional Source Data

This encourages fabrication when the source does not contain the field.

### 3. Treating Schema Validation as Truth Validation

A structurally valid value can still be semantically wrong.

### 4. Retrying Missing Facts

Retries can correct format errors, not invent information that does not exist in the source.

### 5. Unbounded Retry Loops

Use a maximum retry count and escalate after repeated failure.

### 6. Generic Validation Feedback

“Try again” does not tell the model what to fix. Return specific field-level validation errors.

### 7. Using `auto` for Mandatory Extraction

`auto` allows Claude to skip the extraction tool. Use `any` or exact `tool` selection.

## Exam Checklist

- Use tool use or supported schema enforcement for production structured output.
- Use a fake tool when the goal is to force a structured payload rather than perform business logic.
- Use `tool_choice: any` to guarantee at least one tool call.
- Use exact `tool` selection when one extraction tool must run.
- Do not use `auto` for mandatory extraction.
- Keep source fields required but nullable when absence is valid.
- Use `null` instead of fabricated data.
- Distinguish structural errors from semantic errors.
- Use schema validation for format and type failures.
- Use business rules and cross-field checks for interpretation failures.
- Use bounded retries with specific validation feedback.
- Do not expect retries to discover missing facts.
- Escalate after retry exhaustion or unresolved semantic conflict.

## Final Summary

Production-grade extraction needs more than a prompt saying “return JSON.” Use a structured-output tool or schema-enforced tool use, select the required tool with `any` or exact `tool`, and make absent source fields required but nullable.

For the exam, remember: **schemas enforce structure, not truth; retries correct format, not missing information; semantic validation catches wrong interpretations; and `null` is safer than fabricated data.**
