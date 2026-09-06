# Episode 1: Claude Certified Architect

## Overview

Anthropic recently launched the **Claude Certified Architect** certification. This episode introduces the certification, its eligibility requirements, exam format, syllabus, and several important preparation tips.

## Eligibility and Cost

- The exam is currently exclusive to employees of Anthropic partner companies.
- Individual subscribers and the general public cannot register independently at this time.
- Early access is free for the first 5,000 eligible partner-company employees.
- After the early-access allocation is used, the exam costs $99.
- Partner companies may sponsor the certification attempt for their employees.
- Registration should be completed using the official email address associated with the partner company.

## Exam Format

- Duration: **120 minutes**
- Delivery: **Proctored exam**
- Results: Expected in approximately **two days**

## Exam Syllabus

### Agentic Architecture and Orchestration

The exam covers the design of agentic systems, including agent architecture, orchestration patterns, and practical scenario-based decisions.

### Tool Design and MCP Integration

Model Context Protocol (MCP) is an important part of AI application architecture. Candidates should understand how to design tools and integrate MCP with AI applications.

MCP does not replace APIs. Instead, it provides a standardized approach for connecting AI systems to tools and external data sources.

### Claude Code Configuration and Workflows

Candidates should understand how to configure Claude Code and use its workflow features efficiently, including:

- `CLAUDE.md` hierarchy and configuration
- Custom skills
- Custom slash commands
- Project and user-level rules
- Prompt engineering for coding assistants
- Cost and context management

### Prompt Engineering and Structured Output

Prompt engineering is a core skill for AI developers. Preparation should include creating reliable prompts and requesting structured, predictable output from models.

### Context Manageability and Reliability

Candidates should know how to preserve important information during long interactions and avoid exhausting the model's context window.

In Claude Code, the `/compact` command summarizes the current conversation so that important context can be retained while continuing the session.

## Sample Question

### Scenario

You want to create a custom `/review` command that runs your team's standard code-review checklist. The command must be available to every developer when they clone or pull the repository.

### Correct Location

Create the project-scoped command in:

```text
.claude/commands/
```

Commands stored in this directory are version-controlled with the repository and become available to developers when they clone or pull the project.

## Preparation Advice

The exam is expected to test more than definitions. Candidates should prepare with:

- Scenario-based knowledge
- Hands-on Claude Code experience
- Practical understanding of agentic architecture
- Tool and MCP implementation experience
- Familiarity with Claude Code configuration files and workflows
- Experience managing context in long coding sessions

Review the official exam guide and its sample questions before registering. Focus on how each feature is used in a real project, not only on memorizing terminology.

## Key Takeaways

1. The Claude Certified Architect exam is currently limited to Anthropic partner-company employees.
2. Early access is free for the first 5,000 eligible employees; later attempts cost $99.
3. The exam is a 120-minute proctored assessment.
4. Agentic architecture, orchestration, MCP, Claude Code, prompt engineering, structured output, and context management are central topics.
5. Project-scoped custom commands belong in `.claude/commands/` so they can be shared through version control.
6. Practical, hands-on preparation is likely to be more valuable than memorizing isolated facts.