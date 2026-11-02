---
name: mcp-builder
description: Guide for building high-quality MCP (Model Context Protocol) servers and clients — including a concept-first pass that explains MCP before any code — for Python (FastMCP), Node/TypeScript (MCP SDK v1/v2) and Java (Spring Boot with LangChain4j or Spring AI). Use when creating an MCP server to expose an API or service, adding an MCP client such as LangChain4j McpToolProvider to an existing app, or when the user asks what MCP is and whether their project needs a server or a client.
license: Complete terms in LICENSE.txt
---

# MCP Server Development Guide

## Overview

Create MCP (Model Context Protocol) servers that enable LLMs to interact with external services through well-designed tools, and wire MCP clients into existing applications. The quality of an integration is measured by how well it enables LLMs to accomplish real-world tasks.

Two entry points:

- **Explain-only request** ("MCP 是什么", "先解释一下", "这轮不要改代码") → **Step 0**. Concepts only, zero changes.
- **Build request** → Phases 1–4, in the stack the project already uses.

---

## Step 0 — Concept pass (run first when the user asks what MCP is)

1. **Read the official spec, not a scaffold.** Fetch `https://modelcontextprotocol.io/specification/draft.md` (the `.md` suffix returns Markdown; the index is `https://modelcontextprotocol.io/sitemap.xml`).
2. **Fetch the stack-specific pages for the project's language** — in parallel with step 1 when the language is already known. Java / Spring Boot: `https://docs.langchain4j.dev/tutorials/mcp/` and `https://docs.spring.io/spring-ai/reference/api/mcp/mcp-overview.html`. Do not open TypeScript/Python SDK READMEs to answer a Java question.
3. **Explain what MCP is and what problem it solves.** It is a standard interface between AI applications and external capabilities: without it every "AI app × tool" pair needs its own glue code (M × N integrations); with it each side implements the protocol once (M + N). MCP does not prescribe a model, and it does not replace plain HTTP calls.
4. **Separate Function Calling from MCP — they are not the same thing.**
   - Function Calling is a model-side capability: the model emits JSON saying "call `generate_image(prompt=...)`".
   - MCP is the tool-side capability: where the tool schema comes from, who executes it, how arguments travel, how results and errors are returned, and how tools are discovered.
5. **Describe the protocol surface.** JSON-RPC 2.0 underneath; three roles — **host** (the AI application), **client** (the connector inside the host), **server** (the capability provider); `tools/list` for discovery and `tools/call` for invocation, with a parameter schema and a defined error shape. Resources and prompts are part of the protocol too.
6. **Ask which side and which transport before proposing any design.** Use the `ask_user_question` tool with two questions in one call:
   - `id: "side"` — options: expose this project's capability as an MCP server (external agents call you); consume external MCP servers from this project (client, e.g. LangChain4j `McpToolProvider`); both; not decided yet.
   - `id: "transport"` — options: local stdio; remote Streamable HTTP; not decided yet.
   A free-text answer is valid input and is usually more informative than the options.
7. **Treat "还没想好 / 不要改东西 / 先别动代码" as a hard stop.** Close the turn by stating explicitly that no files were changed and no todo was created, then leave the decision questions on the table (who calls it, how async results map onto request/response, how auth works). Do not scaffold, do not write a plan file, do not create a todo.

---

## Process

Creating a high-quality MCP server involves four main phases. Run Step 0 first whenever the user is still deciding what MCP is or which side they need.

### Phase 1: Deep Research and Planning

#### 1.1 Understand Modern MCP Design

**API Coverage vs. Workflow Tools:**
Balance comprehensive API endpoint coverage with specialized workflow tools. Workflow tools can be more convenient for specific tasks, while comprehensive coverage gives agents flexibility to compose operations. Performance varies by client—some clients benefit from code execution that combines basic tools, while others work better with higher-level workflows. When uncertain, prioritize comprehensive API coverage.

**Tool Naming and Discoverability:**
Clear, descriptive tool names help agents find the right tool quickly. Use consistent prefixes (e.g. `github_create_issue`, `github_list_repos`) and action-oriented naming.

**Context Management:**
Agents benefit from concise tool descriptions and the ability to filter/paginate results. Design tools that return focused, relevant data. Some clients support code execution which can help agents filter and process data efficiently.

**Actionable Error Messages:**
Error messages should guide agents toward solutions with specific suggestions and next steps.

#### 1.2 Study MCP Protocol Documentation

**Navigate the MCP specification:**

Start with the sitemap to find relevant pages: `https://modelcontextprotocol.io/sitemap.xml`

Then fetch specific pages with `.md` suffix for markdown format (e.g. `https://modelcontextprotocol.io/specification/draft.md`).

Key pages to review:
- Specification overview and architecture
- Transport mechanisms (streamable HTTP, stdio)
- Tool, resource, and prompt definitions

#### 1.3 Study Framework Documentation

**Default stack (greenfield, no existing project):**
- **Language**: TypeScript (high-quality SDK support and good compatibility in many execution environments e.g. MCPB. Plus AI models are good at generating TypeScript code, benefiting from its broad usage, static typing and good linting tools)
- **Transport**: Streamable HTTP for remote servers, using stateless JSON (simpler to scale and maintain, as opposed to stateful sessions and streaming responses). stdio for local servers.

This default applies only when you are starting from nothing. **When the repo already exists, follow the repo's stack** — a Spring Boot project gets the Java path below, never a TypeScript or Python scaffold next to it.

**Load framework documentation:**

- **MCP Best Practices**: [📋 View Best Practices](./reference/mcp_best_practices.md) - Core guidelines

**For Java (Spring Boot + LangChain4j / Spring AI):**

1. Protocol first: `https://modelcontextprotocol.io/specification/draft.md` (as in 1.2).
2. Client-side wiring: `https://docs.langchain4j.dev/tutorials/mcp/` — the LangChain4j tutorial that covers `McpToolProvider` and how an existing app consumes external MCP servers.
3. Server/client overview: `https://docs.spring.io/spring-ai/reference/api/mcp/mcp-overview.html` — Spring AI's MCP overview and the starter artifacts it names.
4. Read the project's `pom.xml` before naming any dependency. Reuse the client/server starter artifacts the docs name when they are already declared, and keep the versions already on the classpath. Do not invent coordinates or a BOM.
5. Reuse the project's existing infrastructure — its controllers, its auth (e.g. Sa-Token), its async/push channels — instead of standing up a parallel stack.

**For TypeScript (recommended for new servers):**

The TypeScript SDK has two major versions. Pick one before writing code:
- **v2** (`@modelcontextprotocol/server`): the current stable release. Use it for new servers.
- **v1** (`@modelcontextprotocol/sdk`): receives bug and security fixes only. Use it only when the project's `package.json` already depends on `@modelcontextprotocol/sdk`, or migrate that project with `npx @modelcontextprotocol/codemod@latest v1-to-v2 .`

If the project already uses v1, tell the user once, in a sentence or two, that v2 (`@modelcontextprotocol/server`) is the current stable release, that v1 receives only bug and security fixes, and that `npx @modelcontextprotocol/codemod@latest v1-to-v2 .` migrates the project. Then continue with v1 unless the user asks to migrate.

Don't mix the two in one project: v1 code samples (imports from `@modelcontextprotocol/sdk/...`) don't apply to v2.

- **TypeScript SDK (v2)**: Use WebFetch to load `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`; the full docs index is `https://ts.sdk.modelcontextprotocol.io/v2/llms.txt`. If the raw fetch times out once, do not retry it — use the `ts.sdk.modelcontextprotocol.io` docs index instead.
- [⚡ TypeScript Guide (v2)](./reference/node_mcp_server_v2.md) - TypeScript patterns and examples
- For v1 projects only: `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/v1.x/README.md` and the [TypeScript Guide (v1)](./reference/node_mcp_server.md)

**For Python:**
- **Python SDK**: Use WebFetch to load `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`. If that fetch times out, do not retry it; use the Python SDK documentation site that the project's docs link to and cite the page you used. TODO: pin the exact Python SDK docs-site URL the next time a fetch succeeds.
- [🐍 Python Guide](./reference/python_mcp_server.md) - Python patterns and examples

#### 1.4 Plan Your Implementation

**Understand the API:**
Review the service's API documentation to identify key endpoints, authentication requirements, and data models. Use web search and WebFetch as needed.

**Tool Selection:**
Prioritize comprehensive API coverage. List endpoints to implement, starting with the most common operations.

---

### Phase 2: Implementation

#### 2.1 Set Up Project Structure

See language-specific guides for project setup:
- [⚡ TypeScript Guide (v2)](./reference/node_mcp_server_v2.md) - Project structure, package.json, tsconfig.json ([v1](./reference/node_mcp_server.md) for existing v1 projects)
- [🐍 Python Guide](./reference/python_mcp_server.md) - Module organization, dependencies
- **Java**: no scaffold. Add the MCP server and/or client components as beans inside the existing Spring Boot module, following the structure described on the LangChain4j and Spring AI pages fetched in 1.3; keep them out of unrelated business packages.

#### 2.2 Implement Core Infrastructure

Create shared utilities:
- API client with authentication
- Error handling helpers
- Response formatting (JSON/Markdown)
- Pagination support

#### 2.3 Implement Tools

For each tool:

**Input Schema:**
- Use Zod (TypeScript) or Pydantic (Python); on Java, use the schema mechanism of the Spring AI / LangChain4j version you confirmed in 1.3
- Include constraints and clear descriptions
- Add examples in field descriptions

**Output Schema:**
- Define `outputSchema` where possible for structured data
- Use `structuredContent` in tool responses (TypeScript SDK feature)
- Helps clients understand and process tool outputs

**Tool Description:**
- Concise summary of functionality
- Parameter descriptions
- Return type schema

**Implementation:**
- Async/await for I/O operations
- Proper error handling with actionable messages
- Support pagination where applicable
- Return both text content and structured data when using modern SDKs

**Annotations:**
- `readOnlyHint`: true/false
- `destructiveHint`: true/false
- `idempotentHint`: true/false
- `openWorldHint`: true/false

---

### Phase 3: Review and Test

#### 3.1 Code Quality

Review for:
- No duplicated code (DRY principle)
- Consistent error handling
- Full type coverage
- Clear tool descriptions

#### 3.2 Build and Test

**TypeScript:**
- Run `npm run build` to verify compilation
- v2 projects: run the self-check script from the [v2 guide](./reference/node_mcp_server_v2.md#verify-your-server) against the built server
- Test with MCP Inspector: `npx @modelcontextprotocol/inspector`

**Python:**
- Verify syntax: `python -m py_compile your_server.py`
- Test with MCP Inspector

**Java:**
- Compile only the module you touched with the project's own build command (e.g. `mvn -pl <module> -am compile` in a Maven multi-module repo); report the exact command and its result
- Then connect with MCP Inspector (`npx @modelcontextprotocol/inspector`) to list and call the tools

See language-specific guides for detailed testing approaches and quality checklists.

---

### Phase 4: Create Evaluations

After implementing your MCP server, create comprehensive evaluations to test its effectiveness.

**Load [✅ Evaluation Guide](./reference/evaluation.md) for complete evaluation guidelines.**

#### 4.1 Understand Evaluation Purpose

Use evaluations to test whether LLMs can effectively use your MCP server to answer realistic, complex questions.

#### 4.2 Create 10 Evaluation Questions

To create effective evaluations, follow the process outlined in the evaluation guide:

1. **Tool Inspection**: List available tools and understand their capabilities
2. **Content Exploration**: Use READ-ONLY operations to explore available data
3. **Question Generation**: Create 10 complex, realistic questions
4. **Answer Verification**: Solve each question yourself to verify answers

#### 4.3 Evaluation Requirements

Ensure each question is:
- **Independent**: Not dependent on other questions
- **Read-only**: Only non-destructive operations required
- **Complex**: Requiring multiple tool calls and deep exploration
- **Realistic**: Based on real use cases humans would care about
- **Verifiable**: Single, clear answer that can be verified by string comparison
- **Stable**: Answer won't change over time

#### 4.4 Output Format

Create an XML file with this structure:

```xml
<evaluation>
  <qa_pair>
    <question>Find discussions about AI model launches with animal codenames. One model needed a specific safety designation that uses the format ASL-X. What number X was being determined for the model named after a spotted wild cat?</question>
    <answer>3</answer>
  </qa_pair>
<!-- More qa_pairs... -->
</evaluation>
```

---

## Pitfalls

- **`raw.githubusercontent.com` fetches time out on this host.** `web_fetch` on `https://raw.githubusercontent.com/modelcontextprotocol/java-sdk/main/README.md` failed with `tool call timed out after 30000ms`, while `modelcontextprotocol.io/specification/draft.md`, `docs.langchain4j.dev/tutorials/mcp/` and `docs.spring.io/.../mcp-overview.html` all returned HTTP 200. Fetch once; on timeout switch to the framework documentation sites — never retry the raw domain in a loop.
- **Don't offer the TypeScript/Python scaffold to a Java project.** The "recommended stack" is for greenfield JS/Python servers; for Spring Boot answer from the LangChain4j and Spring AI pages and the project's existing module layout, and say which URLs you used.
- **Don't answer a concept question with code or install steps.** A request like "先解释 MCP 是什么" is out of scope for scaffolds, `package.json`/`pom.xml` diffs and SDK installation instructions.
- **Don't bury the Function Calling distinction.** It is the confusion the user actually has; state that Function Calling is model-side and MCP is tool-side.
- **Don't skip the side/transport question.** Server vs client flips the API surface, the auth model, the deployment target and the result semantics; ask before designing.
- **Async capabilities fight request/response.** When the underlying capability is asynchronous (e.g. WebSocket push of a generated image), decide up front between blocking until the result is ready and "submit task → return `taskId` → separate query tool".
- **MCP calls carry no app session.** They do not include the app's login token (e.g. Sa-Token); a remote server needs its own credential.
- **Don't mix TypeScript SDK v1 and v2** in one project — apart from that, v1 samples are dead for v2 code.

---

## Verification

**Concept-only pass (Step 0):**
- The answer states JSON-RPC 2.0, the host/client/server roles, `tools/list` and `tools/call`, and explicitly contrasts Function Calling with MCP.
- It cites the official URLs actually fetched (spec + LangChain4j and/or Spring AI) and contains no TypeScript/Python scaffold or SDK install steps.
- `ask_user_question` was used to ask both the side (`server` vs `client`) and the transport.
- After a "not decided / don't change anything" answer: `git status --porcelain` is empty, no file was written and no todo was created, and the closing message says so.

**Build pass (Phases 2–3):**
- The project's build/compile command succeeds for the touched module.
- MCP Inspector connects, `tools/list` shows the expected tool names, and each tool is called once successfully.
- Error paths return the actionable message you designed, not a stack trace.

---

# Reference Files

## 📚 Documentation Library

Load these resources as needed during development:

### Core MCP Documentation (Load First)
- **MCP Protocol**: Start with sitemap at `https://modelcontextprotocol.io/sitemap.xml`, then fetch specific pages with `.md` suffix
- [📋 MCP Best Practices](./reference/mcp_best_practices.md) - Universal MCP guidelines including:
  - Server and tool naming conventions
  - Response format guidelines (JSON vs Markdown)
  - Pagination best practices
  - Transport selection (streamable HTTP vs stdio)
  - Security and error handling standards

### SDK Documentation (Load During Phase 1/2)
- **Python SDK**: Fetch from `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md` (on timeout, use the SDK docs site instead of retrying)
- **TypeScript SDK (v2)**: Fetch from `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`; full docs index at `https://ts.sdk.modelcontextprotocol.io/v2/llms.txt`
- **TypeScript SDK (v1, existing v1 projects only)**: Fetch from `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/v1.x/README.md`

### Framework Documentation for Java Projects (Load During Step 0 / Phase 1)
- **LangChain4j MCP tutorial**: `https://docs.langchain4j.dev/tutorials/mcp/`
- **Spring AI MCP overview**: `https://docs.spring.io/spring-ai/reference/api/mcp/mcp-overview.html`

### Language-Specific Implementation Guides (Load During Phase 2)
- [🐍 Python Implementation Guide](./reference/python_mcp_server.md) - Complete Python/FastMCP guide with:
  - Server initialization patterns
  - Pydantic model examples
  - Tool registration with `@mcp.tool`
  - Complete working examples
  - Quality checklist

- [⚡ TypeScript Implementation Guide (SDK v2)](./reference/node_mcp_server_v2.md) - Complete TypeScript guide for `@modelcontextprotocol/server` with:
  - Project setup and the server factory
  - Tool design: Zod 4 schemas, output formats, pagination, errors
  - A complete example serving stdio and Streamable HTTP, plus bearer auth
  - Resources, prompts, and notifications
  - A self-check script to verify the built server
  - Links to SDK examples for advanced features
  - A v1 → v2 translation table, migration steps, and a quality checklist

- [⚡ TypeScript Implementation Guide (SDK v1)](./reference/node_mcp_server.md) - The equivalent guide for existing `@modelcontextprotocol/sdk` projects

### Evaluation Guide (Load During Phase 4)
- [✅ Evaluation Guide](./reference/evaluation.md) - Complete evaluation creation guide with:
  - Question creation guidelines
  - Answer verification strategies
  - XML format specifications
  - Example questions and answers
  - Running an evaluation with the provided scripts
