---
type: Web Page
title: Client | Pydantic Docs
resource: https://pydantic.dev/docs/ai/mcp/client
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Client

Pydantic AI can act as an [MCP client](https://modelcontextprotocol.io/quickstart/client), connecting to MCP servers to use their tools as part of an agent run. The `MCPToolset`[toolset](/docs/ai/tools-toolsets/toolsets) wraps the [FastMCP Client](https://gofastmcp.com/clients/) and works with both local (stdio) and remote (Streamable HTTP, SSE) MCP servers.

You need to either install [ pydantic-ai](/docs/ai/overview/install), or 

[with the](/docs/ai/overview/install#slim-install)

`pydantic-ai-slim``mcp` optional group:An [ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) accepts any of the following as its first positional argument:

- A URL string (Streamable HTTP, or SSE if the path ends in `/sse`)
- A path to a local Python or Node.js script (run via stdio)
- A [FastMCP transport](https://gofastmcp.com/clients/transports)like`StdioTransport`,`StreamableHttpTransport`, or`SSETransport`
- A pre-built `fastmcp.Client`(for advanced FastMCP-specific configuration like[OAuth](https://gofastmcp.com/clients/auth/oauth)or[tool transformation](https://gofastmcp.com/patterns/tool-transformation))
- An in-process [FastMCP server](https://gofastmcp.com/servers/)(for testing or single-process deployments — no network round trip)

Each `MCPToolset` instance is a [toolset](/docs/ai/tools-toolsets/toolsets) and can be registered with an [ Agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) via the 

`toolsets` argument.You can use [ async with agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__aenter__) to open and close connections to all registered MCP toolsets (and in the case of stdio servers, start and stop the subprocesses) around the context where they’ll be used in agent runs. You can also use 

`async with toolset` to manage the lifecycle of a specific toolset directly, for example if you’d like to share it across multiple agents. If you don’t explicitly enter one of these context managers, the toolset will be opened and closed automatically as needed.The [Streamable HTTP](https://modelcontextprotocol.io/introduction#streamable-http) transport is the recommended way to connect to a remote MCP server.

Before creating the toolset, we need to run a server that supports the Streamable HTTP transport.

Then we can create the toolset:

Define the MCP toolset with the URL used to connect.

Create an agent with the MCP toolset attached.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

**What’s happening here?**

- The model receives the prompt “What is 7 plus 5?”
- The model decides “Oh, I’ve got this `add`tool, that will be a good way to answer this question”
- The model returns a tool call
- Pydantic AI sends the tool call to the MCP server using the Streamable HTTP transport
- The model is called again with the return value of running the `add`tool (12)
- The model returns the final answer

You can visualise this clearly, and even see the tool call, by adding three lines of code to instrument the example with [logfire](https://logfire.pydantic.dev/docs):

The [HTTP + Server-Sent Events](https://spec.modelcontextprotocol.io/specification/2024-11-05/basic/transports/#http-with-sse) transport is also supported. URLs ending in `/sse` are auto-detected as SSE; for any other path, pass an explicit `SSETransport`.

MCP also offers the [stdio transport](https://spec.modelcontextprotocol.io/specification/2024-11-05/basic/transports/#stdio), where the server is run as a subprocess and communicates with the client over `stdin` and `stdout`. Pass a path to a Python or Node.js script, or build a `StdioTransport` for full control over the command, arguments, and environment.

If you already have a [FastMCP server](https://gofastmcp.com/servers/) in the same Python process as your agent, you can hand it directly to `MCPToolset` and save the network round trip:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Instead of constructing `MCPToolset` instances individually, you can load multiple toolsets from a JSON configuration file using [ load_mcp_toolsets()](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.load_mcp_toolsets).

This is particularly useful when you need to manage multiple MCP servers or want to configure servers externally without modifying code.

The configuration file should be a JSON file with an `mcpServers` object containing server definitions. Each server is identified by a unique key and contains the configuration for that server:

The configuration file supports environment variable expansion using the `${VAR}` and `${VAR:-default}` syntax, [like Claude Code](https://code.claude.com/docs/en/mcp#environment-variable-expansion-in-mcp-json). This is useful for keeping sensitive information like API keys or host names out of your configuration files:

When loading this configuration with [ load_mcp_toolsets()](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.load_mcp_toolsets):

- `${VAR}`references are replaced with the corresponding environment variable values.
- `${VAR:-default}`references use the environment variable value if set, otherwise the default value.

`MCPToolset` accepts a `process_tool_call` callback that lets you customize tool call requests and their responses. A common use case is to inject metadata that the server-side handler needs to read:

How the server reads the injected metadata is MCP server SDK specific. For example, with the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) it’s accessible via the [ ctx: Context](https://github.com/modelcontextprotocol/python-sdk#context) argument on tool handlers:

When connecting to multiple MCP servers that might provide tools with the same name, wrap each `MCPToolset` with [ .prefixed(...)](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.prefixed) to prepend a prefix to its tool names:

MCP servers can provide instructions during initialization that give context about how to best interact with the server’s tools. These are accessible via [ MCPToolset.instructions](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset.instructions) after the connection is established, and can be automatically injected into the agent’s instructions by setting 

`include_instructions=True`:MCP tools can include metadata that provides additional information about the tool’s characteristics, which can be useful when [filtering tools](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FilteredToolset). The `meta` and `annotations` fields can be found on the `metadata` dict on the [ ToolDefinition](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition) object that’s passed to filter functions, and the tool’s output schema (if any) is available as the 

`return_schema` field.[ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) additionally exposes a 

`task: bool` flag indicating whether the server declares support for [task-augmented execution](#background-tasks)on the tool.

[ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) supports MCP 

[task-augmented execution](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/tasks)(SEP-1686). Servers can declare per-tool task support via

`execution.taskSupport`, and `MCPToolset` routes calls accordingly:| `execution.taskSupport` | Behavior | 
|---|---|
| `"required"` | Always calls with `task=True`. The server creates a task and the client awaits the final result via`tasks/result`. | 
| `"optional"` | Always calls with `task=True`to opt in to durability, cancellation, and progress notifications. | 
| `"forbidden"`or absent | Calls normally. | 

For [FastMCP](https://gofastmcp.com/) servers, declare task support per tool with `task=TaskConfig(mode=...)`:

The client side needs no extra configuration — `MCPToolset` sends `task=True` automatically based on the server’s declaration:

MCP servers can provide [resources](https://modelcontextprotocol.io/docs/concepts/resources) — files, data, or content that can be accessed by the client. Resources in MCP are application-driven, with host applications determining how to incorporate context manually based on their needs. They are *not* exposed to the LLM automatically (unless a tool returns a `ResourceLink` or `EmbeddedResource`).

`MCPToolset` exposes methods to discover and read resources:

- `list_resources()`
- `list_resource_templates()`
- `read_resource(uri)`

Text content is returned as `str`, and binary content as [ BinaryContent](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent).

Before consuming resources, we need to run a server that exposes some:

Then we can read them from the client:

*(This example is complete, it can be run “as is”)*

In some environments you need to tweak how HTTPS connections are established — for example to trust an internal Certificate Authority, present a client certificate for **mTLS**, or (during local development only!) disable certificate verification altogether. `MCPToolset` exposes an `http_client` parameter so you can pass your own pre-configured [ httpx.AsyncClient](https://www.python-httpx.org/async/):

When you supply `http_client`, Pydantic AI reuses this client for every request. Anything supported by **httpx** (`verify`, `cert`, custom proxies, timeouts, etc.) therefore applies to all MCP traffic.

When connecting to an MCP server, you can optionally specify an [Implementation](https://modelcontextprotocol.io/specification/2025-11-25/schema#implementation) object as client information that will be sent to the server during initialization. This is useful for:

- Identifying your application in server logs
- Allowing servers to provide custom behavior based on the client
- Debugging and monitoring MCP connections
- Version-specific feature negotiation

## Sampling diagram

Here’s a mermaid diagram that may or may not make the data flow clearer:

```
sequenceDiagram
    participant LLM
    participant MCP_Client as MCP client
    participant MCP_Server as MCP server
    MCP_Client->>LLM: LLM call
    LLM->>MCP_Client: LLM tool call response
    MCP_Client->>MCP_Server: tool call
    MCP_Server->>MCP_Client: sampling "create message"
    MCP_Client->>LLM: LLM call
    LLM->>MCP_Client: LLM text response
    MCP_Client->>MCP_Server: sampling response
    MCP_Server->>MCP_Client: tool call response
```
Pydantic AI supports sampling as both a client and server. See the [server](/docs/ai/mcp/server#mcp-sampling) documentation for details on how to use sampling within a server.

To use sampling as a client, an `MCPToolset` needs to have a [ sampling_model](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset.sampling_model) set. This can be done either directly on the toolset using the 

`sampling_model=` constructor keyword argument, or by using [to use the agent’s model (or one specified as an argument) as the sampling model on all](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.set_mcp_sampling_model)

`agent.set_mcp_sampling_model()``MCPToolset`s registered with the agent.Let’s say we have an MCP server that wants to use sampling (in this case to generate an SVG as per the tool arguments):

## Sampling MCP server

Using this server with an `Agent` will automatically allow sampling:

*(This example is complete, it can be run “as is”)*

In MCP, [elicitation](https://modelcontextprotocol.io/docs/concepts/elicitation) allows a server to request [structured input](https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation#supported-schema-types) from the client for missing or additional context during a session.

Elicitation lets models essentially say “Hold on — I need to know X before I can continue”, rather than requiring everything upfront or taking a shot in the dark.

Elicitation introduces a protocol message type called [ ElicitRequest](https://modelcontextprotocol.io/specification/2025-06-18/schema#elicitrequest), which is sent from the server to the client when it needs additional information. The client can then respond with an 

[or an](https://modelcontextprotocol.io/specification/2025-06-18/schema#elicitresult)

`ElicitResult``ErrorData` message.A typical interaction looks like this:

- User makes a request to the MCP server (e.g. “Book a table at that Italian place”)
- The server identifies that it needs more information (e.g. “Which Italian place?”, “What date and time?”)
- The server sends an `ElicitRequest`to the client asking for the missing information.
- The client receives the request, presents it to the user (e.g. via a terminal prompt, GUI dialog, or web interface).
- User provides the requested information, declines, or cancels.
- The client sends an `ElicitResult`back to the server with the user’s response.
- With the structured data, the server can continue processing the original request.

This allows for a more interactive and user-friendly experience, especially for multi-stage workflows. Instead of requiring all information upfront, the server can ask for it as needed.

To enable elicitation, provide an `elicitation_handler` when creating your `MCPToolset`:

This server demonstrates elicitation by requesting structured booking details from the client when the `book_table` tool is called. Here’s how to wire up the matching client:

MCP elicitation supports string, number, boolean, and enum types with flat object structures only. These limitations ensure reliable cross-client compatibility. See [supported schema types](https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation#supported-schema-types) for details.

MCP elicitation requires careful handling — servers must not request sensitive information, and clients must implement user approval controls with clear explanations. See [security considerations](https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation#security-considerations) for details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/mcp/client
