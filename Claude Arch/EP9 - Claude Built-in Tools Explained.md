# EP09: Claude Built-in Tools Explained

## Lesson Goal

This lesson explains the built-in tools commonly used by Claude Code and coding assistants to explore, understand, modify, and validate a codebase.

The main exam topics are:

- `grep` for searching file contents
- `glob` for finding paths and filenames
- `read` for loading a known file
- `write` for creating or replacing a complete file
- `edit` for targeted changes using a unique anchor
- `bash` for shell commands and diagnostics
- Incremental exploration instead of bulk reading
- The canonical `edit` fallback: read, reason, and write

## Tool Inventory

| Tool | Primary responsibility | Key exam distinction |
| --- | --- | --- |
| `grep` | Search contents | Finds text or symbols inside files |
| `glob` | Search paths and names | Finds files by path or filename pattern |
| `read` | Load a file | Reads a specific known file, usually fully |
| `write` | Create or replace a file | Writes complete file content |
| `edit` | Modify a section | Requires a unique old-string anchor |
| `bash` | Execute shell commands | Runs tests, CLIs, packages, and diagnostics |

Each tool has a clear lane. Using a tool outside its natural purpose can waste tokens, increase risk, and make the workflow less reliable.

## `grep`: Search File Contents

Use `grep` when the question is about what is written inside files.

It can find:

- Function or class definitions
- Function calls
- Imports
- TODO comments
- Configuration values
- Error messages
- Text patterns
- References across a codebase

Examples:

```text
Find every call to process_refund.
Find every file that imports React.
Find all TODO comments.
Find the definition of the order processor.
```

These are content questions, so `grep` is the correct tool.

### What `grep` Does Not Do

`grep` is not primarily a filename or path discovery tool. If the question is “find all TypeScript files,” use `glob`, not `grep`.

### Dedicated `grep` Versus `bash`

A shell command can also run a grep-like search, but a dedicated `grep` tool communicates intent directly and is the expected built-in tool for content search in exam scenarios. Do not default to `bash` for a task that has a purpose-built tool.

## `glob`: Find Paths and Filenames

Use `glob` when the question is about file names, extensions, or directory patterns. It does not inspect the contents of the matched files.

Examples:

```text
Find all Python files in src/orders.
List every TypeScript test file.
Find all Markdown documentation files.
List files under the Terraform directory.
Find every model file.
```

Typical patterns include:

```text
src/**/*.py
src/**/*.test.ts
docs/**/*.md
terraform/**/*.tf
```

### Filing-Cabinet Analogy

- `glob` looks at the labels on folders.
- `grep` opens the folders and searches the documents inside.

## `grep` Versus `glob`

This is a frequent exam distinction.

| Scenario | Correct tool | Reason |
| --- | --- | --- |
| Find every file that imports React | `grep` | The import is file content |
| Find all TypeScript files under `src` | `glob` | The extension is a filename pattern |
| Find every test file | `glob` | The test suffix is a naming pattern |
| Find all calls to `process_refund` | `grep` | The call is code content |
| List files under `terraform` | `glob` | The request is about paths |
| Find all TODO comments | `grep` | The comment text is file content |
| Find a class definition | `grep` | The class declaration is file content |

### Quick Rule

```text
Content question -> grep
Path or filename question -> glob
```

## `read`: Load a Known File

Use `read` when you already know which file is relevant and need to inspect its contents.

Examples:

```text
Read src/orders/processor.py.
Load the constants.py file identified by the previous search.
Read the configuration file before changing it.
```

`read` is different from `grep`:

- `grep` locates matching content across files.
- `read` loads the selected file so the agent can understand it fully.

Typical sequence:

```text
grep: find process_refund
read: inspect the file containing process_refund
```

### Read Cost

Reading a known file is targeted and useful. Reading dozens of unrelated files at the beginning of a task can consume the context window before meaningful reasoning begins.

## `write`: Create or Replace a Complete File

Use `write` when the agent needs to create a new file or replace an entire existing file.

Examples:

```text
Create a new order_status.py endpoint.
Generate a new configuration file from scratch.
Replace a file whose contents need a complete redesign.
```

`write` takes the target path and the complete content. It is a whole-file operation.

### Important Risk

Writing an existing file replaces its previous content. Use it deliberately when a complete replacement is intended.

## `edit`: Make a Targeted Change

Use `edit` when only a specific part of an existing file needs to change.

Conceptually, an edit includes:

```text
file path
old string
new string
```

Example:

```text
Old: const max = 10
New: const max = 50
```

`edit` changes the matching section without rewriting the entire file.

### Why `edit` Is Efficient

For a 500-line file where only three lines must change, `edit`:

- Minimizes the changed surface
- Avoids rewriting unrelated content
- Uses fewer tokens
- Reduces the chance of accidental unrelated changes

## The Unique-Anchor Requirement

The old string used by `edit` must be unique within the target file.

If the exact old string appears multiple times, the tool cannot reliably know which occurrence to replace. The edit may fail or require a more specific anchor.

### Weak Anchor

```text
const max
```

This may appear in several declarations.

### Stronger Anchor

```text
const max = 10;
```

An even longer surrounding block may be appropriate when necessary:

```text
function calculate_refund(amount) {
    const max = 10;
    return Math.min(amount, max);
}
```

Use the smallest reliable unique anchor. More specific context makes the intended edit unambiguous.

## Canonical `edit` Fallback

When `edit` cannot find a unique anchor, use this fallback:

```text
read the full file
    -> reason about the required change
    -> update the content in memory
    -> write the complete updated file
```

This approach is less efficient than a successful targeted edit, but it is reliable because the agent controls the complete resulting content.

### Exam Rule

If an edit fails because the old string appears multiple times, the canonical recovery is **read, modify, and write**, not repeated guessing with the same ambiguous anchor.

## `bash`: Execute Shell Commands

Use `bash` for operations that require a shell, command-line program, or external process.

Examples:

- Run tests
- Run a linter or formatter
- Install packages
- Execute a build
- Call a CLI
- Check Git status
- Run migrations or scripts
- Inspect diagnostics that have no dedicated tool

Examples:

```text
Run the test suite.
Install the project's dependencies.
Run the database migration CLI.
Check the current Git status.
```

Do not use `bash` as the first choice when a dedicated built-in tool directly expresses the operation, such as content search with `grep` or path discovery with `glob`.

## Incremental Exploration

The best general pattern for understanding a codebase is incremental exploration:

```text
glob -> grep -> read -> trace -> grep/read again -> act
```

### Phase 1: Map Paths

Use `glob` to understand the relevant folder structure and locate likely files by pattern.

Example:

```text
Find all Python files under src/orders.
```

### Phase 2: Find Symbols and Content

Use `grep` to locate the entry point, class, function, constant, or configuration key related to the task.

Example:

```text
Find the order-processing entry point and every call to process_refund.
```

### Phase 3: Read Targeted Files

Use `read` on the files identified by the search. Understand the local implementation before changing it.

### Phase 4: Trace Dependencies

The read content reveals related functions, constants, modules, or call sites. Search and read those dependencies as needed.

### Phase 5: Act

- Use `write` for a new file or complete replacement.
- Use `edit` for a targeted unique change.
- Use `bash` to run tests and validation.

## Developer Productivity Example

Suppose the task is:

```text
Understand how orders are processed, create a new status endpoint,
and fix a legacy status-code issue.
```

A disciplined workflow is:

1. `glob`: find Python files under the orders module.
2. `grep`: locate the order-processing class or entry function.
3. `read`: inspect `processor.py`.
4. `grep`: trace constants and notification calls referenced by the processor.
5. `read`: inspect `constants.py` and the notification module.
6. `write`: create the new `order_status.py` endpoint.
7. `edit`: replace the unique legacy status-code declaration.
8. `bash`: run tests and linting.

This approach builds context only as the task requires it.

## Bulk Reading Anti-Pattern

Do not read every file in a large codebase before understanding the task.

Why it fails:

- The context window fills with irrelevant content.
- Important details become harder to retain.
- Token cost increases.
- The agent may reason less accurately.
- No actual implementation work has happened yet.

An exam answer that says “read all files first” is usually a warning sign. Start with targeted discovery and follow the dependency chain.

## Tool Selection Decision Tree

```text
Do you need to find text inside files?
    Yes -> grep

Do you need files by name, extension, or path pattern?
    Yes -> glob

Do you already know the file and need its contents?
    Yes -> read

Are you creating a new file or replacing the whole file?
    Yes -> write

Are you changing a small, uniquely anchored section?
    Yes -> edit

Do you need a shell command, test, package, or CLI?
    Yes -> bash
```

## Exam Scenarios

### Scenario 1: Find Function Calls

**Question:** Find every call to `process_refund` across the repository. Which tool fits?

**Answer:** `grep`, because the function call is file content.

### Scenario 2: Find Test Files

**Question:** Find every `.test.ts` file under `src`. Which tool fits?

**Answer:** `glob`, because the request is based on file path and filename pattern.

### Scenario 3: Understand a Known File

**Question:** A search identified `refunds.py` as the relevant file. What should happen next?

**Answer:** Use `read` to load and inspect that specific file.

### Scenario 4: Create an Endpoint

**Question:** The agent must create a brand-new `order_status.py` file. Which tool fits?

**Answer:** `write`, because the file does not exist and complete content must be generated.

### Scenario 5: Small Existing-File Change

**Question:** Change one unique constant declaration in a 500-line file. Which tool fits?

**Answer:** `edit`, using a unique old-string anchor.

### Scenario 6: Ambiguous Edit

**Question:** The old string occurs three times, so the edit cannot identify the intended occurrence. What is the reliable fallback?

**Answer:** Read the full file, make the intended change in memory, and write the complete updated file.

### Scenario 7: Run Validation

**Question:** Run the project's test suite after making changes. Which tool fits?

**Answer:** `bash`, because this requires a shell command.

### Scenario 8: Efficient Codebase Exploration

**Question:** What is the best first workflow for understanding an unfamiliar billing module?

**Answer:** Use `glob` to map relevant paths, `grep` to find entry points, `read` to inspect targeted files, and then trace dependencies incrementally.

### Scenario 9: Large Codebase

**Question:** An agent reads 50 files before starting any reasoning. What is the architectural problem?

**Answer:** Bulk reading wastes context and tokens. Use incremental exploration and load only relevant files.

## Exam Anti-Patterns

### 1. Using `grep` for Filename Discovery

If the requirement is based on extension or path pattern, use `glob`.

### 2. Using `glob` for Content Search

`glob` does not inspect file contents. Use `grep` for symbols, imports, and comments.

### 3. Reading the Entire Repository

Bulk reading consumes context before the agent has identified relevant files.

### 4. Using `write` for a Tiny Targeted Edit

Replacing an entire file for a small change increases risk. Use `edit` with a unique anchor.

### 5. Repeating an Ambiguous Edit

If the old string is not unique, make the anchor more specific or use the read-plus-write fallback.

### 6. Treating `write` as Non-Destructive

Writing an existing path can replace its complete content. Verify the intended scope before using it.

### 7. Using `bash` for Every Operation

Prefer purpose-built tools for content search, path discovery, and targeted editing.

## Exam Checklist

- `grep` searches file contents.
- `glob` searches file paths and names.
- `read` loads a specific known file.
- `write` creates or completely replaces a file.
- `edit` changes a targeted section without rewriting the full file.
- `edit` requires a unique old-string anchor.
- If `edit` is ambiguous, use read, reason, and write.
- `bash` runs shell commands, tests, CLIs, packages, and diagnostics.
- Use incremental exploration: `glob`, `grep`, `read`, trace, then act.
- Avoid reading large numbers of irrelevant files.
- Prefer dedicated built-in tools over shell equivalents when available.
- Use the smallest tool that directly matches the task.

## Final Summary

Claude's built-in tools each have a distinct responsibility. `grep` finds content, `glob` finds paths, `read` loads known files, `write` creates or replaces complete files, `edit` performs precise unique-anchor changes, and `bash` runs shell operations.

For the exam, remember: **explore incrementally, use the correct tool for the question, and when an edit cannot identify a unique anchor, read the file, make the change, and write the complete updated file.**