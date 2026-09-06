# EP08: MCP Servers, Configuration, Cline and More

## Lesson Goal

This lesson explains how Model Context Protocol (MCP) connects AI applications to external tools, resources, and prompt templates. It also covers project and user configuration, secret handling, Cline integration, and the exam decisions that commonly appear in MCP scenarios.

The main exam topics are:

- MCP purpose and architecture
- Host, client, and server responsibilities
- JSON-RPC communication
- Local STDIO and remote HTTP transports
- Connect, initialize, discover, invoke, and return phases
- Tools, resources, and prompts
- `.mcp.json` versus user-level `.claude.json`
- Environment-variable expansion
- Community versus custom MCP servers
- MCP Inspector and Cline integration
- Tool descriptions, scoping, security, and prompt injection risks

## What Is MCP?

Model Context Protocol is an open protocol for connecting AI applications to external capabilities and context.

MCP can expose:

- Tools that perform actions
- Resources that provide structured content
- Prompt templates that provide reusable instructions
- Metadata that helps an AI application understand available capabilities

Without MCP, tool integrations are often embedded directly in each agent's code. That creates custom, tightly coupled integrations that are difficult to reuse across agents and applications.

With MCP, a server can expose a standard interface that multiple compatible hosts can discover and use.

```text
One MCP server
    -> Agent A
    -> Agent B
    -> Claude Code
    -> Cline
    -> Other compatible AI applications
```

MCP is sometimes compared to USB-C for AI: it standardizes the connection between an AI application and external capabilities.

## MCP Does Not Replace APIs

MCP and APIs solve related but different problems.

| API | MCP |
| --- | --- |
| General service-to-service interface | AI-facing capability and context interface |
| Clients commonly know routes and contracts ahead of time | Compatible clients can discover capabilities |
| Often used by web, mobile, and backend applications | Used by agents and AI applications |
| Exposes business operations or data endpoints | Exposes tools, resources, and prompts with AI-oriented metadata |
| Uses its own transport and contract choices | Commonly uses JSON-RPC over supported transports |

An MCP server may wrap an existing API. The API remains useful behind the server; MCP provides a standardized way for AI applications to discover and use it.

## MCP Architecture

### Host

The host is the AI application or agent that wants to use external capabilities.

Examples:

- A custom agent
- Claude Code
- Cline
- An enterprise assistant

The host owns the overall interaction with the model and may manage one or more MCP clients.

### Client

The MCP client is the connector inside the host. It establishes communication with an MCP server, negotiates capabilities, discovers available items, and sends requests.

An SDK usually provides the client implementation.

### Server

The MCP server exposes tools, resources, prompts, and their metadata. It translates MCP requests into local functions, database operations, API calls, or other backend actions.

```text
AI host or agent
        |
     MCP client
        |
   MCP transport
        |
     MCP server
        |
Backend APIs, databases, files, services
```

The agent does not need to embed every external function directly. It can discover the server's capabilities through the protocol.

## JSON-RPC and Transport

MCP messages use a structured JSON-RPC contract. A simplified request has the following shape:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

The `method` identifies the operation, `id` correlates a response to a request, and `params` contains method input.

MCP messages can travel over different transports.

### STDIO

STDIO is commonly used for a local MCP server.

- The host launches a local process.
- The client communicates through standard input and output.
- The server runs on the same machine.
- Configuration usually specifies a command and arguments.

```text
Host -> launches local process -> STDIO MCP server
```

### Streamable HTTP or HTTP-Based Transport

Remote MCP servers are accessed through a network endpoint.

- The server is hosted separately.
- The client connects to a URL.
- Authentication and network security are required.
- The server may support a session-oriented lifecycle.

```text
Host -> HTTP client -> Remote MCP server
```

The transport choice is driven primarily by where the server runs: local process or remote endpoint. It is not determined by whether the server provides tools or resources.

## MCP Communication Lifecycle

The exact details depend on the transport and client, but the exam-level lifecycle is:

```text
Connect
    -> Initialize
    -> Discover capabilities
    -> Reason about available metadata
    -> Invoke a tool or read a resource
    -> Receive structured results
```

### 1. Connect

The host's MCP client connects to a local process or remote server.

### 2. Initialize

The client and server establish the session and negotiate capabilities. A server may provide protocol and implementation information. Some transports or public servers may not expose a session ID in the same way; do not assume that every client flow visibly returns one.

### 3. Discover

The client asks what the server provides. A common discovery operation is:

```text
tools/list
```

The response contains tool names, descriptions, input schemas, and other metadata. Similar discovery operations can expose resources or prompts.

### 4. Reason

The model receives the available metadata and determines whether a tool or resource is relevant. Tool descriptions strongly influence this decision.

### 5. Invoke or Read

The client sends a tool-call or resource-read request. The MCP server performs the underlying operation.

### 6. Return Structured Results

The server returns a structured response that the client passes to the host and model. The model can then continue its agentic loop or produce a final response.

## Tools, Resources, and Prompts

### Tools

Tools are executable capabilities. They may:

- Query a database
- Send a notification
- Create a ticket
- Search documentation
- Call an existing API
- Update a record

Tools can have side effects and therefore need clear descriptions, input validation, authorization, and approval controls.

### Resources

Resources are pieces of content or context exposed by the server. They are useful for static or catalog-like material such as:

- Project documentation
- Source-code roots
- Policies
- Templates
- Reference files
- Knowledge-base articles

Resources can reduce the need for repeated exploratory searches. Instead of making several search calls to discover a known catalog, the client can navigate a structured resource listing.

### Prompts

Prompts are reusable instruction templates exposed by the server. They can standardize workflows while remaining versionable and discoverable.

The three categories serve different purposes:

| MCP capability | Purpose |
| --- | --- |
| Tool | Perform an operation |
| Resource | Provide content or context |
| Prompt | Provide reusable instructions |

## Project and User Configuration

MCP configuration has different scopes. The key question is:

```text
Does this configuration belong to the project or to the individual user?
```

### Project-Level `.mcp.json`

Place shared project MCP configuration in `.mcp.json` at the repository or project root.

Use it for:

- MCP servers required by the project
- Shared command and argument definitions
- Team-level server names
- Project resources
- Non-secret configuration needed to reproduce the workflow

Because this file is normally version-controlled, every developer can receive the same server configuration when they clone or pull the repository.

Example shape for a local STDIO server:

```json
{
  "mcpServers": {
    "project-docs": {
      "command": "npx",
      "args": ["-y", "project-docs-mcp"]
    }
  }
}
```

Example shape for a remote server:

```json
{
  "mcpServers": {
    "company-docs": {
      "type": "http",
      "url": "https://mcp.example.com/docs",
      "headers": {
        "Authorization": "Bearer ${COMPANY_MCP_TOKEN}"
      }
    }
  }
}
```

The exact fields depend on the client and current MCP configuration schema. Understand the scope and purpose rather than assuming every client accepts identical JSON.

### User-Level `.claude.json`

User-level Claude configuration belongs in the user's home directory, commonly represented as:

```text
~/.claude.json
```

Use it for personal configuration and user-specific settings that should not be committed to the repository.

Examples may include:

- Personal server preferences
- User-specific connection settings
- Local client configuration
- Personal credentials, when the client supports that storage model

The transcript sometimes refers to this as a “cloud” configuration file; the relevant Claude configuration name is `.claude.json`. Verify the current client documentation for the exact supported path and schema.

## Scope Decision Table

| Configuration | Correct location | Version controlled? |
| --- | --- | --- |
| Shared project MCP server definition | Project-root `.mcp.json` | Usually yes |
| Personal preferences | User-level `.claude.json` | No |
| API key or database password | Environment or secret manager | No |
| Shared non-secret resource definition | Project configuration | Usually yes |
| Machine-specific path | User or local configuration | Usually no |

### Common Exam Scenario

**Problem:** MCP tools work on one developer's machine but disappear for teammates after cloning the repository.

**Likely cause:** The configuration was stored only in user-level configuration instead of the shared project `.mcp.json`.

**Fix:** Put the shared server definition in the project-root `.mcp.json` and keep personal values outside version control.

## Environment Variables and Secret Handling

Shared configuration and private credentials must be separated.

Do not commit:

- Database passwords
- GitHub tokens
- Slack webhook URLs
- OAuth secrets
- Private API keys

Use an environment-variable placeholder in shared configuration:

```json
{
  "headers": {
    "Authorization": "Bearer ${GITHUB_MCP_TOKEN}"
  }
}
```

The runtime resolves the value from the local shell, container environment, deployment platform, or secret manager.

```text
Version control: configuration shape and variable name
Runtime secret store: actual credential value
```

The value remains local or managed by the deployment environment, while the project can safely share the configuration contract.

## Local, Community, and Custom Servers

### Community Server

A community server is publicly available and can be reused by many compatible clients. Review its code, permissions, dependencies, and trustworthiness before connecting it to sensitive systems.

### Custom Server

A custom server is built for a specific team, organization, workflow, or private data source.

Use a custom server when:

- Internal documentation must remain private
- Company-specific workflows need AI access
- Existing tools require an organization-specific wrapper
- Authentication and authorization must be controlled internally
- A community server does not provide the required capability

### Local Versus Remote

These are separate decisions:

```text
Who owns the server?  Community or custom
Where does it run?    Local STDIO or remote HTTP
```

A community server can run locally, and a custom server can be remote.

## MCP Inspector and Testing

MCP Inspector is a useful tool for inspecting and testing MCP servers.

It can help verify:

- Connection behavior
- Initialization
- Available tools
- Tool descriptions and schemas
- Resource listings
- Prompt listings
- Tool-call inputs and outputs
- Authentication behavior
- Request history

A manual test flow often resembles:

```text
Connect
    -> Initialize
    -> List tools
    -> Inspect metadata
    -> Call a tool
    -> Inspect the structured result
```

Public servers may simplify or hide session details, while private servers may require authentication and explicit session handling.

## Cline and Coding-Assistant Integration

Cline and other coding assistants can act as MCP hosts or clients. They can connect to local or remote MCP servers and use the exposed tools and resources during development.

Typical integration steps are:

1. Add a server definition in the coding assistant's MCP settings.
2. Choose local STDIO or remote HTTP configuration.
3. Configure authentication without exposing secrets in source control.
4. Connect or restart the server.
5. Inspect the discovered tools and metadata.
6. Enable approval controls appropriate to the tool risk.
7. Use the tools through the assistant and inspect the request history when debugging.

Do not automatically approve all tools for a sensitive server. Approval settings should reflect the side effects and trust level of each tool.

## Tool Descriptions in MCP

MCP tools compete for the model's attention alongside built-in tools such as file search or shell tools. Vague MCP descriptions may cause the model to choose a built-in tool instead.

Weak:

```text
Tool to search the codebase.
```

Strong:

```text
Perform semantic code search by intent across the indexed repository.
Use this for concept-based queries when exact text matching is insufficient.
Return ranked file paths and matching excerpts.
Do not use for exact literal matching; use the built-in text search instead.
```

The strong description defines purpose, invocation criteria, output, and the alternative tool boundary.

The first fix for incorrect MCP tool selection is usually a clearer, more specific description. Use hooks or policy gates when the problem is a deterministic authorization or safety requirement.

## Tool Visibility and Context Pollution

Do not assume that tools are lazily loaded only when needed. Depending on the host and client, the model may receive a large tool inventory and its descriptions in context.

Too many tools can cause:

- Context pollution
- Higher prompt size and cost
- Lower routing confidence
- More overlapping descriptions
- More difficult debugging

Use role-based tool scoping:

```text
Research agent: search_web, fetch_url, extract_content
Document agent: read_document, parse_pdf, extract_sections
Synthesis agent: verify_fact
```

Expose only the tools each agent needs. This improves routing and supports least privilege.

## Security and Trust

MCP standardizes communication; it does not automatically secure the underlying system.

Continue to use appropriate:

- Authentication
- Authorization
- OAuth or signed credentials
- Secret managers
- TLS and network controls
- Tenant isolation
- Input validation
- Audit logging
- Approval workflows

### Prompt Injection and Tool Poisoning

An untrusted server or resource may provide malicious instructions, misleading descriptions, or unsafe tool behavior. Treat server metadata and returned content as untrusted input.

Risks include:

- Prompt injection through resource content
- Tool poisoning through misleading descriptions
- Exfiltration of sensitive context
- Unauthorized side effects
- Malicious or compromised community servers

Mitigations include:

- Use trusted and reviewed servers.
- Limit permissions and tool visibility.
- Require approval for consequential actions.
- Validate inputs and outputs.
- Avoid sending secrets or unnecessary context.
- Monitor and audit tool calls.
- Keep credentials outside configuration files and source control.

## Exam Decision Framework

| Scenario | Best answer |
| --- | --- |
| Shared MCP definition missing for teammates | Add it to project-root `.mcp.json` |
| Personal secret or credential | Environment variable or secret manager |
| Local MCP process | STDIO configuration with command and arguments |
| Hosted MCP service | Remote HTTP configuration with secure authentication |
| Static documentation catalog | Expose MCP resources |
| Internal company data | Custom authenticated MCP server |
| Community capability already available | Review and use a trusted community server |
| Wrong custom tool selected | Improve the tool description and boundaries |
| Too many tools visible to one agent | Scope tools by role |
| Need to inspect server behavior | Use MCP Inspector or equivalent testing tool |
| MCP replacing all backend APIs | Incorrect; MCP complements and can wrap APIs |

## Exam Scenarios

### Scenario 1: Tools Missing After Clone

**Question:** An agent works locally, but teammates cannot discover its MCP tools after cloning the project. What is the likely fix?

**Answer:** Put the shared MCP server definitions in the project-root `.mcp.json`. Do not rely only on a user-level `.claude.json` file.

### Scenario 2: Secret in Version Control

**Question:** A project configuration needs a GitHub token, but the repository is shared. How should it be configured?

**Answer:** Commit a variable placeholder such as `${GITHUB_MCP_TOKEN}` and provide the actual value through the runtime environment or secret manager.

### Scenario 3: Local or Remote Transport

**Question:** An MCP server runs as a local process on the developer's machine. Which transport configuration is appropriate?

**Answer:** STDIO with the command and arguments needed to launch the local process.

### Scenario 4: Static Documentation

**Question:** An agent repeatedly searches hundreds of stable documentation pages to find known project references. What MCP capability can reduce exploratory calls?

**Answer:** Expose the documentation catalog as MCP resources so the agent can navigate structured content directly.

### Scenario 5: Internal Company Tools

**Question:** A coding assistant needs access to private internal documentation that must not be uploaded to a public model service. What architecture fits?

**Answer:** Use a private, authenticated MCP server behind the organization's security controls and configure the assistant to access it securely.

### Scenario 6: MCP and APIs

**Question:** Should an organization replace all APIs with MCP servers?

**Answer:** No. MCP complements APIs and can wrap existing API operations for AI-facing discovery and use.

### Scenario 7: Wrong Tool Selection

**Question:** The model always chooses a built-in exact-search tool instead of a custom semantic-search MCP tool. What should be changed first?

**Answer:** Improve the custom tool description so it clearly explains semantic intent search, output, and when it should be preferred over exact matching.

### Scenario 8: Dangerous Community Server

**Question:** A community MCP server requests broad filesystem and credential access. What should the architect do?

**Answer:** Review or reject the server, reduce permissions, require approval, and avoid exposing secrets or broad capabilities without a justified trust model.

## Exam Anti-Patterns

### 1. Putting Shared Configuration in User Scope

User-level configuration works only for that user and machine. Shared project configuration belongs in the repository's `.mcp.json`.

### 2. Committing Credentials

Never hardcode tokens, passwords, or webhook secrets in `.mcp.json` or source code.

### 3. Treating MCP as a Replacement for APIs

MCP is an AI-facing integration protocol. Existing APIs may remain behind the MCP server.

### 4. Exposing Every Tool to Every Agent

Use role-based tool scope to reduce context pollution and enforce least privilege.

### 5. Using Vague Tool Descriptions

Tool metadata drives model selection. Explain purpose, input, output, boundaries, and alternatives.

### 6. Trusting Community Servers Automatically

Community availability is not proof of safety. Review code, permissions, network access, and side effects.

### 7. Assuming MCP Provides Security by Itself

Authentication, authorization, secret management, validation, and auditing remain application responsibilities.

## Exam Checklist

- Know that MCP standardizes AI access to tools, resources, and prompts.
- Distinguish host, client, and server responsibilities.
- Understand JSON-RPC request structure and request IDs.
- Recognize STDIO for local processes and HTTP-based transport for remote servers.
- Know the connect, initialize, discover, invoke, and return lifecycle.
- Understand that session details can vary by transport and server implementation.
- Put shared project server definitions in project-root `.mcp.json`.
- Keep personal configuration in user-level `.claude.json` where supported.
- Keep secrets in environment variables or a secret manager.
- Use MCP resources for structured, relatively static context catalogs.
- Distinguish community servers from custom authenticated internal servers.
- Use MCP Inspector to inspect tools, schemas, resources, and calls.
- Improve MCP tool descriptions before adding unnecessary prompting.
- Scope tools by agent role to reduce context pollution.
- Remember that MCP complements rather than replaces APIs.
- Treat server metadata and returned content as potentially untrusted.
- Apply authentication, authorization, approval, and audit controls.

## Final Summary

MCP provides a standard way for AI hosts to discover and use external tools, resources, and prompts. The host uses an MCP client to connect to a local or remote server, initialize communication, discover metadata, invoke capabilities, and receive structured results.

For the exam, remember: **shared project configuration belongs in `.mcp.json`, private values belong in runtime secret management, local servers commonly use STDIO, remote servers use network transport, and MCP complements APIs rather than replacing them.**