# EP10: `CLAUDE.md` Hierarchy and Config Rules

## Lesson Goal

This lesson explains how Claude Code loads persistent instructions and how to choose the correct scope for shared conventions, personal preferences, directory-specific guidance, and conditional file rules.

The main exam topics are:

- Persistent context through `CLAUDE.md`
- User, project, and directory scope
- Project-root versus `.claude/CLAUDE.md`
- Modular instructions with `@import`
- `.claude/rules/` files and YAML front matter
- Conditional rules using `paths`
- Glob patterns for rule targeting
- `/memory` for configuration diagnostics
- Common hierarchy and sharing mistakes

## The Missing-Instructions Problem

An agent can appear familiar with a project after one developer has worked with it for weeks. A new teammate who clones the same repository may start with none of that project knowledge unless the instructions are stored in a shared location.

Without persistent instructions, the new session may need to rediscover:

- Technology choices
- Code style
- Testing conventions
- Architecture decisions
- Error-handling rules
- Tool usage expectations
- Project-specific terminology

That rediscovery consumes context, time, and tokens. `CLAUDE.md` provides persistent project context that Claude Code can load before working in a session.

## What Is `CLAUDE.md`?

`CLAUDE.md` is a briefing document for Claude Code. It can describe the project's technology stack, conventions, architecture, workflows, and constraints.

Typical content includes:

- How to install and test the project
- Directory and module structure
- Naming and formatting conventions
- Preferred libraries and patterns
- Error-handling conventions
- Agent-loop rules
- Security or deployment constraints
- Commands that should be used or avoided

It is persistent context, not a conversation message. A new session can load the instructions without requiring the developer to repeat them manually.

## The Three Main Scopes

Claude Code instructions can be organized at three practical levels:

```text
User level       -> Personal defaults across projects
Project level    -> Shared repository conventions
Directory level  -> Local guidance for a subtree
```

Each scope has a different audience and sharing behavior.

## User-Level Instructions

User-level instructions live in the user's home configuration area and apply across projects for that user.

Conceptually:

```text
~/.claude/CLAUDE.md
```

Use this scope for personal preferences such as:

- Preferred explanation style
- Personal workflow preferences
- Default commit-message style
- Personal productivity conventions
- User-specific instructions that should not be imposed on a team

User-level instructions are not part of the repository. When another developer clones the project, those instructions do not appear in the developer's home directory.

### Exam Rule

If a question says that a rule must be shared with teammates or loaded in CI, user-level instructions are the wrong location.

## Project-Level Instructions

Project-level instructions belong inside the repository and are shared through version control.

Two common project locations are:

```text
CLAUDE.md
.claude/CLAUDE.md
```

Keeping `CLAUDE.md` at the repository root makes it highly discoverable. Keeping the file inside `.claude` organizes Claude-specific project configuration in a dedicated directory. Teams should choose the layout supported by their Claude Code version and local conventions.

Use project scope for:

- Shared architecture
- Repository setup commands
- Team coding conventions
- Test and build instructions
- Shared error categories
- Required agent-loop behavior
- Project-specific tool guidance

Project-level instructions should be committed to the repository when they are intended for the team. Teammates and CI can then load the same shared context after cloning or pulling the project.

## Directory-Level Instructions

Directory-level instructions apply to a directory and its relevant subtree. They provide more specific guidance for a part of the project.

Examples:

```text
infrastructure/CLAUDE.md
packages/api/CLAUDE.md
packages/frontend/CLAUDE.md
```

Use directory-level files when different parts of the repository have different conventions:

- Terraform conventions under `infrastructure/`
- API conventions under `packages/api/`
- React component rules under `packages/frontend/`
- Data-pipeline conventions under `data/`

Directory-level guidance is more surgical than a project-wide file and should stay close to the code it describes.

## Scope Comparison

| Scope | Typical location | Audience | Shared through repository? |
| --- | --- | --- | --- |
| User | Home configuration | One developer across projects | No |
| Project | Root `CLAUDE.md` or `.claude/CLAUDE.md` | Whole project team | Yes, when committed |
| Directory | A project subdirectory | Contributors to that area | Yes, when committed |
| Rules | `.claude/rules/` with optional paths | Files matching conditions | Yes, when committed |

## Scope Selection Questions

1. Is this a personal preference across projects? Use user-level instructions.
2. Does the entire repository need this rule? Use project-level `CLAUDE.md`.
3. Does only one subsystem need this guidance? Use directory-level `CLAUDE.md`.
4. Should the rule apply only to particular file patterns? Use `.claude/rules/` with `paths` front matter.

## Modular Instructions with `@import`

A large `CLAUDE.md` file can become difficult to maintain. Use `@import` to compose focused instruction files.

Example:

```markdown
# Project Instructions

@import ./docs/api-patterns.md
@import ./docs/testing-conventions.md
@import ./docs/deployment-rules.md
```

Good import candidates include API patterns, testing conventions, deployment rules, security guidance, database conventions, and domain glossaries.

Benefits include smaller focused files, clear ownership, cleaner pull requests, fewer merge conflicts, and easier maintenance.

### Exam Rule

If the question asks how to make a large `CLAUDE.md` modular, choose `@import` rather than duplicating instructions in every directory.

## `.claude/rules/` Directory

The `.claude/rules/` directory contains topic-specific rule files. Rules are useful when guidance should apply conditionally based on file paths.

Example layout:

```text
.claude/
  CLAUDE.md
  rules/
    typescript.md
    react-testing.md
    terraform.md
```

Unlike broad project instructions, rules can target matching files through front matter.

## YAML Front Matter and `paths`

A rule file can include front matter such as:

```yaml
---
description: TypeScript conventions
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
---

# TypeScript Rules

- Do not use `any`.
- Prefer `unknown` with type narrowing.
- Annotate function parameters and return types.
```

The front matter describes when and why the rule applies.

### `description`

The description identifies the rule's purpose.

```yaml
description: React component testing patterns
```

### `paths`

The `paths` field makes the rule conditional by specifying which file patterns trigger it.

```yaml
paths:
  - "**/*.test.tsx"
  - "**/*.spec.ts"
```

This prevents a React testing rule from being applied to unrelated infrastructure or backend files.

## Glob Pattern Reference

Glob patterns determine which files a rule targets.

### `**`

Matches any number of directory levels.

```text
terraform/**/*.tf
```

This targets Terraform files at any depth under `terraform/`.

### `*`

Matches a name or segment within a directory level.

```text
src/api/*.ts
```

This targets TypeScript files directly inside `src/api/`, not necessarily deeper nested directories.

### Common Patterns

| Pattern | Meaning |
| --- | --- |
| `terraform/**/*.tf` | All Terraform files at any depth under `terraform` |
| `**/*.test.tsx` | Any `test.tsx` file anywhere |
| `**/*.spec.ts` | Any `spec.ts` file anywhere |
| `src/api/*.ts` | TypeScript files directly inside `src/api` |
| `src/**/*.{ts,tsx}` | TypeScript and TSX files at any depth under `src` |

### Exam Rule

When a question says that a rule must apply only to certain file types or patterns, use `.claude/rules/` with YAML front matter and `paths`.

## `CLAUDE.md` Versus `.claude/rules/`

| Requirement | Recommended mechanism |
| --- | --- |
| General project briefing | Project `CLAUDE.md` |
| Personal default across projects | User-level `CLAUDE.md` |
| Guidance for one subsystem | Directory-level `CLAUDE.md` |
| Conditional rules by filename or path | `.claude/rules/` with `paths` |
| Split a large instruction file | `@import` |
| Debug what loaded in a session | `/memory` |

A directory-level `CLAUDE.md` is naturally tied to a location. A rule file with `paths` is conditionally tied to matching file patterns and can target patterns across the repository.

## `/memory` Diagnostic Command

Use:

```text
/memory
```

to inspect what memory and instruction files Claude Code has loaded for the current session.

It can help diagnose:

- A project instruction file that was not loaded
- A rule file with a non-matching path pattern
- An incorrectly named file
- Unexpected user-level instructions
- Scope or configuration mistakes

### Exam Rule

If the question asks how to verify which `CLAUDE.md` files or rules are active in a session, choose `/memory`.

## Team Configuration Failure Scenario

### Problem

An original developer has excellent project conventions in Claude Code. A new teammate clones the repository, but Claude behaves as if it has never seen the project.

### Likely Cause

The instructions were stored only in the original developer's home directory.

### Fix

Move shared project instructions into a committed project-level `CLAUDE.md` or `.claude/CLAUDE.md`. Keep genuinely personal preferences at user scope.

### Diagnostic

Run `/memory` to confirm which instruction files are loaded.

## Conditional TypeScript and Terraform Example

Suppose a repository needs TypeScript rules for `src/**/*.ts` and `src/**/*.tsx`, and Terraform rules for all files under `terraform/`.

Use separate rule files:

```text
.claude/rules/typescript.md
.claude/rules/terraform.md
```

`typescript.md`:

```yaml
---
description: TypeScript conventions
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
---

- Do not use `any`.
- Use explicit function parameter and return types.
```

`terraform.md`:

```yaml
---
description: Terraform conventions
paths:
  - "terraform/**/*.tf"
---

- Use the repository's required provider version.
- Run formatting and validation before submitting changes.
```

This keeps both rule sets conditional and focused.

## Exam Scenarios

### Scenario 1: New Teammate Lacks Conventions

**Question:** A developer's personal Claude Code session knows the project rules, but teammates do not after cloning the repository. What should be checked first?

**Answer:** Check the instruction scope. Shared project rules should be in a committed project-level `CLAUDE.md` or `.claude/CLAUDE.md`, not only in the original developer's home directory.

### Scenario 2: Personal Preferences

**Question:** A developer wants verbose commit messages in every project, but does not want to impose that preference on teammates. Where should it go?

**Answer:** User-level instructions in the home configuration area.

### Scenario 3: Backend-Only Guidance

**Question:** API conventions should apply only while working under `packages/api`. What should be used?

**Answer:** A directory-level `CLAUDE.md` inside the API directory, or a path-targeted rule if the condition is based on file patterns.

### Scenario 4: File-Type Conditional Rules

**Question:** Testing rules must apply to all `*.test.tsx` and `*.spec.ts` files across the repository. What is the best configuration?

**Answer:** A `.claude/rules/` file with YAML front matter containing a `paths` list for those glob patterns.

### Scenario 5: Large Instruction File

**Question:** The root `CLAUDE.md` has become thousands of lines long and multiple teams need to maintain separate sections. What mechanism helps?

**Answer:** Split focused documents and compose them with `@import`.

### Scenario 6: Rules Not Loading

**Question:** A developer suspects that a rule file is not active in the current session. How can they inspect loaded instructions?

**Answer:** Run `/memory`.

### Scenario 7: Terraform Pattern

**Question:** Terraform guidance should apply to every `.tf` file at any depth under `terraform/`. Which pattern fits?

**Answer:** `terraform/**/*.tf` in the rule file's `paths` front matter.

## Exam Anti-Patterns

### 1. Storing Team Rules Only at User Scope

User-level instructions are not automatically shared when the repository is cloned.

### 2. Putting Personal Preferences in the Repository

Do not impose one developer's personal workflow on the entire team unless it is genuinely a project requirement.

### 3. Using a Broad `CLAUDE.md` for Conditional Rules

When rules depend on file paths or extensions, use `.claude/rules/` with `paths` rather than cramming every condition into one global briefing file.

### 4. Using Directory Prefixes as a Substitute for `paths`

A directory-level file and a path-targeted rule have different scope semantics. Use front matter when the rule must match file patterns across directories.

### 5. Allowing One Monolithic Instruction File to Grow Unchecked

Use `@import` to keep instructions modular and maintainable.

### 6. Guessing What Loaded

Use `/memory` to inspect active memory and configuration rather than assuming the session loaded a file.

## Exam Checklist

- `CLAUDE.md` provides persistent context for Claude Code.
- User-level instructions apply to one developer across projects and are not shared automatically.
- Project-level instructions belong in the repository and should be committed when shared.
- Directory-level instructions specialize guidance for a subtree.
- Project instructions can commonly live at the root or under `.claude/`.
- Use `@import` to split large instruction files into focused modules.
- Use `.claude/rules/` for topic-specific conditional rules.
- Use YAML front matter with `description` and `paths`.
- `paths` uses glob patterns to target matching files.
- `**` matches any number of directory levels.
- Use `/memory` to diagnose what instructions are loaded.
- Move shared rules from user scope into project scope when teammates need them.

## Final Summary

`CLAUDE.md` is persistent context for Claude Code. User-level files provide personal defaults, project-level files provide shared repository guidance, and directory-level files provide local specialization. Use `@import` for modular composition and `.claude/rules/` with `paths` front matter for conditional file-pattern rules.

For the exam, remember: **shared team guidance belongs in the committed project scope, personal preferences belong in user scope, conditional file rules belong in `.claude/rules/`, and `/memory` shows what the session actually loaded.**
