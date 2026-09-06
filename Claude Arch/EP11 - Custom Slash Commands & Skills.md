# EP11: Custom Slash Commands and Skills

## Lesson Goal

This lesson explains how to create reusable Claude Code workflows with custom slash commands and skills, and how to choose the correct scope, isolation, and tool permissions for exam scenarios.

The main exam topics are:

- Custom slash commands as reusable Markdown prompts
- Project-scoped versus user-scoped commands
- Commands versus `CLAUDE.md`
- Skills as on-demand mini-agents
- Skill anatomy: name, context, allowed tools, and argument hint
- `context: fork` for isolated execution
- `allowed-tools` and least privilege
- Templates and reference files
- Avoiding context pollution

## Custom Slash Commands

A custom slash command packages reusable instructions into a Markdown file that can be invoked with a slash command.

Instead of repeatedly typing:

```text
Run all tests in this module and summarize failures with file names and line numbers.
```

you can store the workflow in a command file and invoke it as:

```text
/test
```

A command is more than simple text expansion. It contains a prompt or instruction set that Claude reads and executes when the command is invoked.

## Command Location and Scope

The directory where a command is stored determines who can use it.

### Project-Scoped Commands

Store team commands in the project repository:

```text
.claude/commands/review.md
```

These commands can be committed to version control and shared with every developer who clones or pulls the repository.

Use project scope for:

- Team code-review checklists
- Security-review workflows
- Shared pull-request templates
- Project test commands
- Release checklists
- Repository-specific diagnostics

### User-Scoped Commands

Store personal commands in the user's home configuration area:

```text
~/.claude/commands/review.md
```

Use user scope for commands that should be available across the user's projects but should not be imposed on the team.

Examples:

- Personal pre-commit checklist
- Personal cross-project research workflow
- Personal explanation or documentation style
- A private productivity macro

### Scope Comparison

| Scope | Typical location | Distribution | Best for |
| --- | --- | --- | --- |
| Project | `.claude/commands/` in repository | Team and CI when committed | Shared project workflows |
| User | `~/.claude/commands/` | One user across projects | Personal workflows |

### Exam Signal

If the question says **available to all contributors**, store the command in the project's `.claude/commands/` directory and commit it to the repository.

If the question says **personal across projects**, use the user's home-level commands directory.

## Commands Versus `CLAUDE.md`

These mechanisms have different jobs.

### `CLAUDE.md`

`CLAUDE.md` contains persistent project context and ambient rules. Claude reads it as part of understanding the project and its conventions.

Use it for:

- Technology stack
- Repository architecture
- Coding conventions
- Testing rules
- General safety constraints
- Shared project instructions

### Slash Command

A slash command is an explicitly invoked reusable procedure.

Use it for:

- A code-review checklist
- A repeatable security scan
- A release process
- A frequently used test-and-report workflow
- A task-specific prompt sequence

| Feature | `CLAUDE.md` | Custom slash command |
| --- | --- | --- |
| Activation | Ambient or session initialization | Explicit slash invocation |
| Purpose | Persistent project context and rules | Reusable task procedure |
| Example | Project coding conventions | `/review` |
| Execution style | Applies broadly | Runs when requested |

Do not put a long, rarely needed procedure into `CLAUDE.md`, because it will pollute every session's context. Move complex on-demand procedures into a skill or command.

## What Is a Skill?

A skill is an on-demand, reusable capability that behaves like a configured mini-agent. It can have its own context, tools, constraints, templates, and reference material.

A skill is more adaptive than a static command:

- It can ask clarifying questions.
- It can use selected tools.
- It can follow a multi-step procedure.
- It can use supporting templates and references.
- It can run in the main session or a forked context.
- Claude can invoke it based on the user's request when the skill is relevant.

A skill is not necessarily a separately spawned sub-agent, but it can run in an isolated fork when configured to do so.

## Skill Location and Structure

Skills live under a dedicated skills directory. Each skill should have its own folder and a required `SKILL.md` file.

Project example:

```text
.claude/
  skills/
    create-agent/
      SKILL.md
      templates/
        single-agent.py
        multi-agent.py
      references/
        agent-guidelines.md
```

Another skill can be organized as:

```text
.claude/
  skills/
    deep-review/
      SKILL.md
      references/
        security-checklist.md
```

The required file is `SKILL.md`. Supporting templates and references are optional, but useful for complex workflows.

## Skill Anatomy

A production skill commonly defines four important pieces of metadata or behavior:

1. `name`
2. `context`
3. `allowed-tools`
4. `argument-hint`

The exact front-matter schema can vary by Claude Code version, so verify current syntax against the official documentation. The exam concepts remain the same.

### `name`

Identifies the skill and explains its purpose.

```yaml
name: create-agent
```

Use a clear name that matches the capability rather than a vague label.

### `context`

Controls where the skill runs.

```yaml
context: fork
```

A forked context runs the skill separately from the main conversation. Without fork isolation, the skill runs in the current main session.

### `allowed-tools`

Defines the tools the skill can use.

```yaml
allowed-tools:
  - Read
  - Grep
  - Glob
```

Grant only the tools the skill needs. This follows least privilege and reduces accidental tool use.

### `argument-hint`

Describes an argument the user should provide or helps Claude request a missing argument.

```yaml
argument-hint: path to file or directory
```

Use it when a skill needs a target file, directory, issue ID, or another required input.

## Example Skill Metadata

Conceptually:

```yaml
---
name: deep-review
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: file or directory to review
---

Perform a security-focused code review.

1. Identify the target from the user's argument.
2. Inspect relevant files incrementally.
3. Check authentication, authorization, input validation, and data exposure.
4. Report findings with severity, evidence, and remediation.
5. Return only the structured review summary to the main session.
```

The skill body contains the procedure. Supporting files can provide detailed checklists and reusable templates.

## `context: fork`

A skill configured with `context: fork` runs in a separate context instead of consuming the main conversation's working context.

```text
Main session
    -> invokes skill
    -> forked skill context performs work
    -> formatted findings return to main session
```

### When to Use a Fork

Use `context: fork` when:

- The skill analyzes many files.
- The procedure has many intermediate steps.
- Intermediate reasoning should not flood the main conversation.
- The main session contains valuable context that should remain focused.
- The work can be performed independently and summarized afterward.

### What Fork Does Not Mean

Forking is not a permission boundary. It controls execution context and isolation. It does not determine which tools the skill can use.

`allowed-tools` controls capabilities; `context: fork` controls where execution occurs.

## `allowed-tools` and Least Privilege

A skill should receive only the tools required for its task.

Examples:

```text
Read-only review skill: Read, Grep, Glob
Agent-scaffolding skill: Read, Write, Edit, Glob
Test-reporting skill: Read, Bash
```

Do not give a read-only security-review skill permission to modify files unless modification is part of its explicit responsibility.

### Why Tool Restriction Matters

- Reduces accidental side effects
- Makes behavior easier to reason about
- Reduces tool-selection ambiguity
- Supports least privilege
- Limits exposure to sensitive operations
- Helps make the skill reusable and auditable

## `argument-hint`

Use `argument-hint` when a skill needs a target or parameter.

Examples:

```yaml
argument-hint: path to file
argument-hint: directory to analyze
argument-hint: issue number
```

If the user does not provide the required argument, the skill can ask for it rather than guessing.

`argument-hint` does not:

- Isolate execution
- Restrict tools
- Make a command project-scoped
- Replace the skill's task instructions

## Supporting Skill Files

A skill can include more than `SKILL.md`.

### Templates

Templates provide boilerplate or known structures.

Examples:

```text
templates/single-agent.py
templates/multi-agent.py
templates/tool-definition.json
```

They reduce repeated generation and make output more consistent.

### References

Reference files provide background knowledge, rules, or detailed checklists.

Examples:

```text
references/agent-guidelines.md
references/security-checklist.md
references/output-schema.md
```

The skill's `SKILL.md` should explain when to consult these files.

### Required Versus Optional

- `SKILL.md` is required.
- Templates are optional.
- Reference files are optional.
- Supporting files are useful when the workflow is complex or needs stable boilerplate.

## Command Versus Skill

| Feature | Custom slash command | Skill |
| --- | --- | --- |
| Main form | Markdown prompt file | Folder containing `SKILL.md` |
| Activation | Explicit `/command` | On demand based on task, or through a command/workflow |
| Complexity | Static reusable procedure | Multi-step adaptive capability |
| Own context | Usually main session | Main session or `context: fork` |
| Tool permissions | Command instructions | Explicit `allowed-tools` when configured |
| Supporting files | Usually simple | Templates and references supported |
| Best for | Repeatable fixed prompt | Complex procedure or mini-agent |

### Decision Rule

- Use `CLAUDE.md` for ambient project rules.
- Use a slash command for a fixed, explicitly invoked prompt.
- Use a skill for a complex, adaptive, multi-step procedure.
- Use `context: fork` when the skill's intermediate work should be isolated.
- Use `allowed-tools` to limit capability.
- Use `argument-hint` when a target input is required.

## Example: Project Review Command

A team wants a standard review command available to every developer.

```text
.claude/commands/review.md
```

Possible content:

```markdown
Review the current changes for:

1. Correctness and regressions
2. Security issues
3. Missing tests
4. Error handling
5. Breaking API changes

Report findings by severity with file paths and recommended fixes.
```

Because the file is in project scope and committed, every developer can invoke `/review` after cloning the repository.

## Example: Deep Review Skill

A team wants an in-depth security review that may inspect hundreds of files without flooding the main conversation.

```text
.claude/skills/deep-review/SKILL.md
```

Conceptual configuration:

```yaml
---
name: deep-review
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: file or directory
---
```

The skill can search and read relevant files incrementally, apply a security checklist from a reference file, and return only a structured finding summary to the main session.

## Context Pollution

Context pollution occurs when every session receives instructions or intermediate results that are relevant only to an occasional task.

Examples:

- Putting a 50-step security audit in `CLAUDE.md`
- Running a large review skill in the main session when its intermediate reasoning is not needed there
- Giving every command and skill every available tool
- Loading every template and reference document for every task

Reduce pollution by:

- Keeping ambient rules concise
- Moving complex procedures into skills
- Using `context: fork` for large independent work
- Scoping `allowed-tools`
- Loading references only when needed

## Exam Scenarios

### Scenario 1: Team Slash Command

**Question:** A `/review` command must be available to every developer after cloning the repository. Where should it be stored?

**Answer:** In the project repository's `.claude/commands/` directory and committed to version control.

### Scenario 2: Personal Command

**Question:** A developer has a private cross-project checklist that should be available only to them. Where should it be stored?

**Answer:** In the user's home-level commands directory.

### Scenario 3: Static Workflow

**Question:** A fixed prompt should run the same security checklist whenever a developer invokes it manually. Which mechanism fits?

**Answer:** A custom slash command.

### Scenario 4: Complex Adaptive Procedure

**Question:** A workflow needs to inspect many files, ask clarifying questions, use templates, and return structured findings. Which mechanism fits?

**Answer:** A skill.

### Scenario 5: Intermediate Results Flood the Main Session

**Question:** A skill analyzes hundreds of files, and the team does not want intermediate reasoning to consume the main conversation context. What should change?

**Answer:** Configure the skill with `context: fork`.

### Scenario 6: Skill Permission Scope

**Question:** A read-only review skill should inspect code but never modify it. How should this be enforced?

**Answer:** Restrict its `allowed-tools` to read and search tools such as `Read`, `Grep`, and `Glob`, excluding write or edit tools.

### Scenario 7: Missing Target Input

**Question:** A skill requires a file path, but users often forget to provide one. Which metadata helps the skill request the missing value?

**Answer:** `argument-hint`.

### Scenario 8: Project Rules Versus Procedure

**Question:** A long deployment procedure is placed in `CLAUDE.md` and consumes context in every session. What is the better design?

**Answer:** Move the on-demand procedure into a command or skill. Keep `CLAUDE.md` for concise ambient project rules.

### Scenario 9: Fork Versus Permission

**Question:** A team wants to isolate a skill's intermediate work and restrict its access to read-only tools. Which settings address each requirement?

**Answer:** Use `context: fork` for isolation and `allowed-tools` for capability restriction.

## Exam Anti-Patterns

### 1. User Scope for Team Commands

A command in the user's home directory is not automatically available to teammates. Use project scope for team distribution.

### 2. Putting Procedures in `CLAUDE.md`

`CLAUDE.md` is ambient project context. Do not fill it with long workflows that are needed only occasionally.

### 3. Treating Skills as Simple Text Macros

Skills can have context isolation, tool permissions, arguments, templates, and references. They are more capable than static command text.

### 4. Confusing Fork with Tool Restriction

`context: fork` isolates execution. `allowed-tools` restricts capabilities. They solve different problems.

### 5. Giving a Skill Excessive Tools

Use least privilege. A skill should not receive write, shell, or network access unless it needs those capabilities.

### 6. Running Large Skills in the Main Session

Use a fork for expensive independent analysis when intermediate results would pollute the main context.

### 7. Using `argument-hint` for Isolation

An argument hint helps obtain missing input; it does not create a separate session or restrict tools.

## Exam Checklist

- Slash commands are reusable Markdown instruction files.
- Project commands live in `.claude/commands/` and can be version-controlled.
- User commands live in the user's home commands directory and are personal.
- “Available to all contributors” means project scope.
- `CLAUDE.md` contains ambient project rules and persistent context.
- Skills live in their own folders under `.claude/skills/`.
- Each skill requires a `SKILL.md` file.
- Skills can include templates and reference files.
- `context: fork` isolates skill execution from the main session.
- `allowed-tools` controls the skill's capabilities.
- `argument-hint` helps obtain required target input.
- Use skills for complex adaptive procedures.
- Use commands for fixed explicitly invoked workflows.
- Keep long on-demand procedures out of ambient `CLAUDE.md` context.
- Apply least privilege to skill tools.

## Final Summary

Custom slash commands package fixed reusable prompts, while skills provide richer on-demand capabilities with their own instructions, tools, arguments, supporting files, and optional forked context.

For the exam, remember: **project commands belong in `.claude/commands/`, personal commands belong in user scope, complex procedures belong in skills, `context: fork` isolates execution, `allowed-tools` restricts capabilities, and `argument-hint` supplies missing inputs.**
