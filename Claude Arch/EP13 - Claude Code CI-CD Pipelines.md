# EP13: Claude Code CI/CD Pipelines

## Lesson Goal

This lesson explains how to run Claude Code safely and reliably inside an automated CI/CD pipeline.

The main exam topics are:

- Interactive versus non-interactive execution
- The `-p` and `--print` flags
- Machine-readable JSON output
- JSON schema enforcement
- Fresh-session review isolation
- Prior findings and review deduplication
- `CLAUDE.md` for shared testing and review standards
- Automated merge gates
- Batch APIs and blocking-workflow risks

## The CI/CD Problem

Claude works naturally in an interactive terminal because a developer can answer follow-up questions. A CI/CD runner has no human available to answer clarification requests.

```text
Interactive local session:
Claude asks a question -> Developer answers -> Claude continues

Automated CI job:
Claude asks a question -> Nobody answers -> Job hangs or times out
```

A workflow that works locally can therefore fail in GitHub Actions or another CI environment if Claude is still expecting an interactive user.

## Non-Interactive Execution

Use the print mode for automation:

```text
claude -p "Review the changed files and report critical issues."
```

The long form is:

```text
claude --print "Review the changed files and report critical issues."
```

The `-p` or `--print` option runs Claude in a non-interactive mode suitable for scripts and CI/CD jobs. It allows the command to produce a result without waiting for a human conversation.

### Exam Rule

When a question asks which option enables non-interactive Claude execution in CI, choose `-p` or `--print`.

Do not choose fabricated alternatives such as:

- `--headless`
- `--batch`
- `--ci`
- `--non-interactive`

The exam signal is the print flag.

## Non-Interactive Design Requirements

A non-interactive CI prompt should:

- State the task clearly.
- Include all required context.
- Avoid requiring clarification.
- Define what to do when information is missing.
- Specify the output format.
- Define failure and exit behavior.
- Limit the scope of files and operations.

Example:

```text
Review only the files changed in this pull request.
Do not ask follow-up questions.
If evidence is insufficient, report the issue as unresolved.
Return only JSON matching the supplied schema.
```

The command must be deterministic enough for an unattended runner.

## Human-Readable Versus Machine-Readable Output

Default prose is useful for developers but difficult for downstream pipeline steps to parse reliably.

### Human Output

```text
I found a critical authentication issue in src/auth.ts on line 42.
```

### Machine Output

```json
{
  "findings": [
    {
      "severity": "critical",
      "file": "src/auth.ts",
      "line": 42,
      "message": "Authentication check can be bypassed."
    }
  ]
}
```

Use structured output when a later CI step must:

- Parse findings
- Block a merge
- Create an issue
- Generate an artifact
- Route a notification
- Calculate a status
- Make a deployment decision

## Output Format and JSON Schema

Tell Claude both the output format and the exact shape expected by downstream automation.

Conceptually:

```text
claude -p "Review the pull request and return findings." \
  --output-format json \
  --json-schema review-schema.json \
  > findings.json
```

The exact CLI syntax can vary by Claude Code version, so verify current documentation. The exam concept is:

```text
Non-interactive print mode + machine-readable output + schema validation
```

### Why Use a Schema?

A schema:

- Makes the contract explicit.
- Reduces parsing ambiguity.
- Helps downstream jobs validate the response.
- Prevents free-form prose from silently breaking automation.
- Supports reliable severity and location fields.

Example schema concept:

```json
{
  "type": "object",
  "required": ["findings"],
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["severity", "file", "line", "message"],
        "properties": {
          "severity": {"enum": ["critical", "high", "medium", "low"]},
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "message": {"type": "string"}
        }
      }
    }
  }
}
```

## Session Isolation for CI Reviews

A Claude session that wrote code may carry reasoning that biases how it reviews that code. It may remember why a design was chosen and unconsciously defend it instead of challenging it.

A fresh review session provides a more independent perspective:

```text
Build or implementation session
        |
        v
Fresh review session
        |
        v
Fresh test or integration-review session
```

### Why Fresh Sessions Help

- The reviewer does not inherit the implementation rationale.
- The review starts from the current repository state.
- The model is less likely to rationalize its own earlier decisions.
- Different CI stages have clearer responsibilities.

### Recommended Pattern

Use separate sessions for separate jobs:

1. Build or generate code.
2. Start a fresh session for code review.
3. Start another fresh session for test analysis if needed.
4. Pass only the required artifacts and shared project instructions.

A fresh session does not mean no context. The session can receive `CLAUDE.md`, changed-file information, prior findings, and explicit task input.

## `CLAUDE.md` in CI/CD

Use project-level `CLAUDE.md` to standardize what every automated session should know.

Useful CI/CD instructions include:

- Required test commands
- Test naming conventions
- Mocking standards
- Coverage targets
- Security-review criteria
- Severity definitions
- Output expectations
- Deployment restrictions
- Review rules
- Commands that must not be run in production

Every fresh session can load the same committed project guidance without duplicating it in every pipeline prompt.

### Example Project Rules

```markdown
# CI Review Rules

- Run the targeted test suite before the full suite.
- Treat authentication and authorization regressions as critical.
- Return findings with file, line, severity, and remediation.
- Do not modify files during review jobs.
- Exit with a failure status when critical findings exist.
```

## Review Deduplication

Fresh sessions do not remember previous review comments. Without a handoff mechanism, Claude may report the same finding on every new commit.

### Deduplication Workflow

```text
1. Fetch the current pull request and changed files.
2. Retrieve prior review findings or comments.
3. Store them in a file or structured variable.
4. Start a fresh non-interactive Claude session.
5. Pass prior findings as context.
6. Ask Claude to report only new or changed findings.
7. Save the new structured findings.
```

Example prompt:

```text
Review the current pull request changes.
These findings were already reported in an earlier review:

<prior findings>

Do not repeat unchanged findings. Reassess them only if the new commits
change their status. Report only new findings or findings whose severity,
location, or remediation has materially changed.
Return JSON matching the review schema.
```

### Why This Matters

Without prior findings, each fresh session sees the pull request as new and may repeat identical comments. Passing prior findings reduces duplicate work, token cost, and review noise.

## Merge Gates

Structured findings can drive automated decisions.

```text
Claude review
    -> findings.json
    -> Parse severity
    -> Critical finding?
        Yes -> Block merge
        No  -> Continue pipeline
```

Example shell logic:

```text
if jq -e '.findings[] | select(.severity == "critical")' findings.json; then
  exit 1
fi
```

The exact command depends on the CI platform and tools available. The architectural pattern is what matters: validate structured output, then make an explicit gate decision.

### Important Boundary

Do not let free-form prose directly control a deployment or merge decision. Validate the output schema and apply deterministic pipeline logic.

## Testing Standards with `CLAUDE.md`

A vague command such as:

```text
Generate tests for this class.
```

may produce tests that do not match the project's conventions.

Project instructions can define:

- Test directory structure
- Naming patterns
- Mocking rules
- Fixture conventions
- Required negative cases
- Coverage expectations
- Integration-test requirements
- How failures should be reported

Then every automated test-generation session inherits the same standards.

## Production Pipeline Architecture

A robust Claude review job can follow this structure:

```text
Project CLAUDE.md
        |
        v
Load changed files and prior findings
        |
        v
Fresh Claude session with -p/--print
        |
        v
JSON output validated against schema
        |
        v
Parse severity and changed findings
        |
        v
Block or continue merge
```

### Pipeline Responsibilities

- `CLAUDE.md`: Shared conventions and review policy
- Prior-finding step: Deduplication context
- `-p` or `--print`: Non-interactive execution
- JSON and schema: Machine-readable contract
- Fresh session: Independent review context
- CI script: Deterministic merge or deployment decision

## Batch APIs and Blocking Workflows

A batch API may be useful for asynchronous, non-blocking work, but it is not appropriate when a pipeline must receive a result immediately before deciding whether to merge or deploy.

### Do Not Use a Long-Latency Batch for

- Blocking pre-merge checks
- Immediate deployment approvals
- Synchronous security gates
- Steps that require a result before the next job

If a service has a long processing window and no latency guarantee, it should not be the dependency for a blocking workflow.

### Better Uses

- Offline analysis
- Non-blocking reporting
- Large asynchronous evaluations
- Periodic quality dashboards
- Backlog classification

## Security and Operational Controls

Running Claude in CI can expose source code and pipeline context. Apply controls such as:

- Use least-privilege CI credentials.
- Limit the files and commands Claude can access.
- Avoid exposing secrets in prompts or logs.
- Use read-only review jobs when possible.
- Require approval for modifications or deployments.
- Validate generated output before acting on it.
- Pin or review action and dependency versions.
- Record tool calls and pipeline artifacts for auditability.
- Set explicit timeouts and bounded retries.

Non-interactive does not mean unsupervised. It means the workflow must provide all required decisions and controls programmatically.

## Exam Scenarios

### Scenario 1: Local Works, CI Times Out

**Question:** A Claude command works in a developer's terminal but hangs in GitHub Actions. What is the likely cause and fix?

**Answer:** The command is waiting for interactive clarification. Run it with `-p` or `--print`, provide complete instructions, and define behavior for missing information.

### Scenario 2: Downstream Step Cannot Parse Output

**Question:** Claude returns prose, but the next CI step expects structured findings. What should change?

**Answer:** Request machine-readable JSON and enforce the expected shape with a JSON schema.

### Scenario 3: Review Repeats Comments

**Question:** A new CI review session reports the same findings on every commit. What should the pipeline do?

**Answer:** Fetch prior review findings, pass them into the fresh session, and ask Claude to report only new or materially changed findings.

### Scenario 4: Self-Review Bias

**Question:** Why should a review job use a fresh session instead of the session that generated the code?

**Answer:** A fresh session provides independent context and is less likely to rationalize or defend its earlier implementation decisions.

### Scenario 5: Test Standards

**Question:** Generated tests use inconsistent names and mocks across CI runs. What should standardize them?

**Answer:** Put project testing conventions, naming patterns, mocking rules, and coverage requirements in a committed project-level `CLAUDE.md`.

### Scenario 6: Critical Finding Gate

**Question:** The pipeline must block merging when Claude identifies a critical issue. What is the reliable design?

**Answer:** Request schema-validated JSON, parse the severity deterministically, and exit the CI job with failure when a critical finding exists.

### Scenario 7: Batch Processing

**Question:** A batch API may take up to 24 hours and has no immediate latency guarantee. Is it suitable for a blocking pre-merge check?

**Answer:** No. Use a synchronous or suitably bounded workflow for a gate that must decide before merging.

## Exam Anti-Patterns

### 1. Running Interactive Claude in CI

A runner cannot answer follow-up questions. Use `-p` or `--print` and make the prompt self-contained.

### 2. Relying on Default Prose

Downstream machines should not parse free-form prose. Use JSON and schema validation.

### 3. Reusing the Implementation Session for Review

The same reasoning context can bias review. Use a fresh session for independent analysis.

### 4. Omitting Prior Findings

Fresh sessions do not remember old comments. Pass prior findings to prevent duplicate review output.

### 5. Using Claude's Prose as a Direct Merge Gate

Parse and validate structured output before taking a deterministic pipeline action.

### 6. Using Long-Latency Batches for Blocking Jobs

A batch without a latency guarantee cannot safely control an immediate merge or deployment decision.

### 7. Exposing Secrets in Prompts or Configuration

Use CI secret stores and environment variables. Do not place credentials in committed prompt files or logs.

## Exam Checklist

- Use `-p` or `--print` for non-interactive CI execution.
- Make automated prompts complete and avoid clarification dependencies.
- Request JSON when a downstream machine must consume the result.
- Use a JSON schema to enforce the output contract.
- Use fresh sessions for independent build, review, and test jobs.
- Use an explore or summary handoff when prior context is needed without carrying the full session.
- Fetch and pass prior review findings to deduplicate comments.
- Put shared test and review standards in project-level `CLAUDE.md`.
- Parse structured severity and apply deterministic merge gates.
- Do not use high-latency batch processing for blocking workflows.
- Keep CI credentials and sensitive source context protected.
- Set explicit timeouts and bounded retries.

## Final Summary

CI/CD runners have no human available to answer Claude's follow-up questions. Use `-p` or `--print` for non-interactive execution, return schema-validated JSON for downstream automation, and use fresh sessions for independent review stages. Pass prior findings to fresh sessions to prevent duplicate comments, and use project-level `CLAUDE.md` to standardize testing and review behavior.

For the exam, remember: **non-interactive CI means `--print`, machine workflows need JSON and schemas, fresh sessions improve review independence, prior findings prevent duplicate work, and long-latency batches should not block merges.**
