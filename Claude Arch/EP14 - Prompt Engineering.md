# EP14: Prompt Engineering, Explicit Criteria and False Positives

## Lesson Goal

This lesson explains how to write prompts that produce focused, reliable results instead of vague, noisy, or fabricated output.

The main exam topics are:

- Vague instructions and false positives
- Categorical criteria
- Explicit include and skip rules
- Severity definitions
- Structured output requirements
- Trust collapse in automated review
- Disabling noisy categories
- Prompt separation by concern
- Preventing fabricated data in extraction workflows

## The False-Positive Problem

Suppose a Claude-powered code reviewer reports dozens of findings on every pull request. Some are real security issues, but many are style opinions, naming suggestions, theoretical edge cases, or optimizations that the project does not care about.

After repeated noisy reviews, developers stop reading the output. A real SQL injection vulnerability can then be ignored because it appears among dozens of low-value findings.

```text
Too many false positives
    -> Developers lose trust
    -> Developers ignore the reviewer
    -> Real issues are missed
```

False positives are not merely annoying. They can create a safety problem by destroying trust in every category of output.

## Why Vague Instructions Fail

A prompt such as:

```text
Be thorough and conservative in your analysis.
```

sounds reasonable but does not define what should be reported. The model must invent its own meaning of “thorough” and “conservative.” It may report:

- Variable naming preferences
- Formatting opinions
- Minor performance ideas
- Theoretical edge cases
- Suggestions not enforced by the project's linter
- Micro-optimizations with no practical impact

The model does not know the team's definition of a meaningful issue from adjectives alone.

### Exam Rule

When a question asks how to reduce false positives, reject vague instructions such as:

- Be conservative
- Be thorough
- Use your best judgment
- Report only important issues

Replace them with explicit categorical criteria.

## Why Confidence Thresholds Fail

Another tempting prompt is:

```text
Only report issues you are 80% confident about.
```

This does not reliably align the model's confidence with the team's standards.

The model's confidence is influenced by its training distribution, not by the project's definition of a valid finding. It may be highly confident about a naming convention and less confident about a subtle security flaw, even though the security flaw matters far more to the team.

A confidence score is not a calibrated project policy.

### Exam Rule

Do not use model confidence thresholds as the primary way to define review precision. Tell the model exactly what to report and what to skip.

## Categorical Criteria: The Correct Fix

Categorical criteria replace subjective judgment with explicit rules.

Instead of:

```text
Be conservative and report important issues.
```

Write:

```text
Report only:
- SQL injection, XSS, or CSRF vulnerabilities.
- Missing error handling on critical payment paths.
- Race conditions in shared-state operations.
- Hard-coded credentials.
- Weak encryption implementations.
- Significant logic errors that can cause incorrect business behavior.

Do not report:
- Variable naming preferences.
- Formatting already enforced by the linter.
- Comment style.
- Minor performance improvements below the agreed threshold.
- Personal refactoring preferences.
- Theoretical issues without a credible execution path.
```

The model receives a rulebook instead of being asked to invent one.

## Anatomy of an Explicit Review Prompt

A precise review prompt should define:

1. Role and task
2. Categories to report
3. Categories to skip
4. Severity mapping
5. Evidence requirements
6. Scope boundaries
7. Output format
8. Behavior when evidence is insufficient

### Example

```text
Review only the changed files in this pull request.

Report findings only when they match one of these categories:
- SQL injection, XSS, or CSRF
- Authentication or authorization bypass
- Missing error handling on critical paths
- Race conditions involving shared state
- Hard-coded credentials or weak encryption
- Significant logic errors that can produce incorrect business results

Do not report:
- Naming or formatting preferences
- Linter-approved style
- Comment preferences
- Minor refactoring opportunities
- Speculative issues without a reproducible path
- Performance improvements under five milliseconds

For every finding, include evidence from the changed code.
If the evidence is insufficient, omit the finding rather than guessing.
Return only the required JSON array.
```

## Severity Criteria

Do not rely only on vague severity words such as “serious” or “high priority.” Define what each level means.

Example:

```text
Critical:
- A directly exploitable security vulnerability.
- A payment, authentication, or authorization failure with a credible impact.
- A defect that can corrupt production data.

High:
- A significant logic error on a critical path.
- Missing error handling that can cause failed transactions.
- A race condition that can produce inconsistent state.

Medium:
- A reproducible issue with limited scope or a practical workaround.

Low:
- A real correctness or maintainability issue that does not create immediate operational risk.
```

Severity should be based on explicit project criteria and evidence, not on an uncalibrated model confidence number.

## Structured Output

Automated systems should not rely on free-form prose. Define an output schema that downstream tools can validate.

Example:

```json
[
  {
    "severity": "critical",
    "category": "sql_injection",
    "file": "src/db/users.ts",
    "line": 42,
    "message": "User input is concatenated into a SQL query.",
    "evidence": "query += userInput",
    "remediation": "Use a parameterized query."
  }
]
```

A schema can require:

- Severity
- Category
- File
- Line
- Message
- Evidence
- Remediation

Structured output makes it possible to sort findings, create tickets, block merges, and avoid parsing ambiguous prose.

## Separate Concerns

Do not combine unrelated review objectives into one vague prompt. Separate concerns by category or pipeline stage.

Examples:

- Security review
- Correctness review
- Test coverage review
- Performance review
- Documentation review

Each stage can have its own criteria and output contract. This reduces category overlap and makes the results easier to trust.

### Pipeline Example

```text
Security review -> security findings
Correctness review -> logic findings
Test review -> missing-test findings
        |
        v
Coordinator or CI aggregator
```

Separate prompts are especially useful when a noisy category needs to be tuned or temporarily disabled without affecting security or correctness checks.

## Disabling a Noisy Category

Sometimes one category has a persistently high false-positive rate despite prompt improvements. Continuing to emit noisy output can damage trust in the entire reviewer.

A valid temporary strategy is:

1. Disable the problematic category.
2. Keep reliable categories active.
3. Measure and improve the disabled category separately.
4. Add clearer criteria and examples.
5. Re-enable it after validation.

### Do Not Lower the Threshold as a First Fix

If a category is already producing false positives, lowering a confidence threshold does not create a project-specific definition of correctness. It may continue producing unreliable findings.

### Exam Rule

When a noisy category is causing trust collapse, temporarily disabling that category is often safer than continuing to report unreliable results or merely changing a confidence threshold.

## Extraction Prompts and Fabricated Data

The same principle applies to structured data extraction. A model may fill missing fields by inference unless the prompt explicitly forbids fabrication.

This is dangerous in medical, insurance, financial, and compliance workflows.

### Unsafe Instruction

```text
Extract the patient information from the report.
```

The model may infer a patient ID, diagnosis code, or dosage that does not appear in the source.

### Safer Instruction

```text
Extract only information explicitly present in the source document.
- Preserve patient IDs and diagnosis codes exactly as written.
- If a field is absent, return null.
- If a field is ambiguous, return the raw text and mark it ambiguous.
- Never infer a dosage, diagnosis, identifier, or other medical fact.
- Do not normalize an uncertain value to the closest known value.
```

### Null Is Valid

A null value is an honest representation of missing information. A fabricated value can create legal, medical, financial, or operational harm.

## Explicit Constraints

Strong prompts define both positive and negative behavior.

### Positive Criteria

```text
Always extract the patient ID when it appears.
Return ICD-10 codes exactly as written.
Return the source page for each extracted field.
```

### Negative Criteria

```text
Never fabricate missing values.
Do not infer a diagnosis from symptoms.
Do not invent a dosage.
Do not silently resolve ambiguous text.
```

Positive and negative criteria reduce the space of possible interpretations.

## Before and After Review Design

### Before

```text
Be thorough and conservative. Report anything suspicious.
```

Possible result: 47 findings, most of which are style preferences or speculation.

### After

```text
Report only SQL injection, XSS, CSRF, authentication bypass,
critical-path error handling failures, race conditions, hard-coded secrets,
and significant logic errors.
Skip style, naming, linter-approved formatting, minor optimizations,
and speculative issues without evidence.
Return only schema-valid findings with file, line, category, severity,
evidence, and remediation.
```

Possible result: a smaller set of actionable findings, such as one injection flaw, one missing error handler, and one race condition.

This is not merely “less aggressive” reviewing. It is better-defined reviewing.

## Exam Decision Framework

| Problem | Best first response |
| --- | --- |
| Too many vague findings | Add categorical include and skip rules |
| Model reports naming opinions | Explicitly exclude style and naming categories |
| Confidence values do not match team priorities | Stop relying on confidence thresholds; define severity and evidence |
| Output cannot be consumed by a pipeline | Require a structured schema |
| One category creates most false positives | Temporarily disable that category and improve it separately |
| Missing fields are being invented | Require null for absent data and prohibit inference |
| Several review concerns are mixed together | Separate prompts or pipeline stages by concern |

## Exam Scenarios

### Scenario 1: Noisy Code Review

**Question:** A reviewer reports 50 issues per pull request, including naming opinions and real security defects. What should be done first?

**Answer:** Replace vague instructions with categorical criteria that explicitly list what to report and what to skip.

### Scenario 2: “Be Conservative”

**Question:** Will “be conservative and thorough” reliably reduce false positives?

**Answer:** No. These adjectives do not define the project's standards. Use explicit categories, exclusions, evidence requirements, and output rules.

### Scenario 3: Confidence Threshold

**Question:** Will “only report issues with 80% confidence” calibrate the reviewer to the team's priorities?

**Answer:** No. Model confidence is not calibrated to the project. Define what counts as a reportable issue instead.

### Scenario 4: Security and Style Categories

**Question:** Developers ignore the reviewer because a style category generates many false positives, including alongside security findings. What is a safe temporary action?

**Answer:** Disable the noisy style category while retaining reliable security categories, then improve the disabled category separately.

### Scenario 5: Medical Extraction

**Question:** A patient ID is missing from a medical report. What should the extraction agent return?

**Answer:** `null` or the explicitly defined missing-value representation. It must not infer or fabricate an ID.

### Scenario 6: Ambiguous Diagnosis Code

**Question:** The source contains an ambiguous diagnosis code. What should happen?

**Answer:** Preserve the raw text and mark it ambiguous, rather than silently normalizing it to a guessed code.

### Scenario 7: Machine Consumer

**Question:** A downstream CI step needs severity, file, line, and remediation. What should the prompt require?

**Answer:** A structured JSON output with a schema containing those fields.

## Exam Anti-Patterns

### 1. Vague Adjectives

“Be thorough,” “be conservative,” and “use your best judgment” leave the model to invent criteria.

### 2. Confidence as Policy

A model confidence score is not a substitute for team-defined categories and severity rules.

### 3. Reporting Everything Suspicious

Suspicion without a defined category and evidence creates noise and trust collapse.

### 4. Lowering a Noisy Category's Threshold

A lower threshold does not fix a category whose definition is unclear. Temporarily disable it and improve its criteria.

### 5. Mixing All Review Concerns

Security, style, performance, and correctness often need separate criteria and sometimes separate pipeline stages.

### 6. Fabricating Missing Data

Never fill an absent medical, financial, or identity field by inference when the task is extraction.

### 7. Free-Form Output in Automation

Require structured output when another machine or pipeline stage must consume the result.

## Exam Checklist

- Replace vague prompt adjectives with categorical criteria.
- Explicitly list what to report.
- Explicitly list what not to report.
- Define evidence requirements and severity levels.
- Use project-specific rules rather than model confidence thresholds.
- Separate review concerns when their criteria differ.
- Temporarily disable a category that causes trust-damaging false positives.
- Require structured output for downstream automation.
- Return null for missing extraction fields.
- Preserve ambiguous source text instead of guessing.
- Never fabricate medical, financial, identity, or compliance data.
- Remember that reducing noise protects trust in valid findings.

## Final Summary

Reliable prompt engineering tells the model exactly what to report, what to skip, how to classify severity, and what format to return. Vague instructions and confidence thresholds do not reliably reduce false positives because they ask the model to invent project-specific judgment.

For the exam, remember: **use categorical criteria first, separate concerns, require evidence and structured output, disable persistently noisy categories when necessary, and return null rather than fabricating missing data.**
