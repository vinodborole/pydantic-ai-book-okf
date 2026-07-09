---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/mcp/overview
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Overview

Pydantic AI supports [Model Context Protocol (MCP)](https://modelcontextprotocol.io) in multiple ways:

- [Agents](/docs/ai/core-concepts/agent)can connect to MCP servers and use their tools — see- [Use MCP servers](#use-mcp-servers)below.
- Agents can be used within MCP servers. [Learn more](/docs/ai/mcp/server)

The Model Context Protocol is a standardized protocol that allows AI applications (including programmatic agents like Pydantic AI, coding agents like [cursor](https://www.cursor.com/), and desktop applications like [Claude Desktop](https://claude.ai/download)) to connect to external tools and services using a common interface.

As with other protocols, the dream of MCP is that a wide range of applications can speak to each other without the need for specific integrations.

There is a great list of MCP servers at [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers).

Some examples of what this means:

- Pydantic AI could use a web search service implemented as an MCP server to implement a deep research agent
- Cursor could connect to the [Pydantic Logfire](https://github.com/pydantic/logfire-mcp)MCP server to search logs, traces and metrics to gain context while fixing a bug
- Pydantic AI, or any other MCP client could connect to our [Run Python](https://github.com/pydantic/mcp-run-python)MCP server to run arbitrary Python code in a sandboxed environment

The recommended way to give an agent access to an MCP server is the [ MCP capability](/docs/ai/core-concepts/capabilities#mcp). It runs the MCP server locally by default — keeping credentials, hooks, and tracing under your control — and lets you opt into the model provider’s 

[native MCP support](/docs/ai/overview/native-tools#mcp-server-tool)with a single

`native=True` flag, so the same agent works across providers without code changes:Pass a URL as the first argument to enable both the local fallback and (with `native=True`) provider-native MCP. On the local side, `local=` accepts any [ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) input — a URL, FastMCP transport, pre-built 

`fastmcp.Client`, in-process `FastMCP` server, or local script path. See the [capability documentation](/docs/ai/core-concepts/capabilities#mcp)for the full set of inputs and configuration options.

For lower-level access — managing the toolset lifecycle directly, sharing one MCP server across multiple agents, or passing advanced transport / client configuration that doesn’t fit the capability shape — use `MCPToolset` directly via `toolsets=[...]`. See the [MCP client documentation](/docs/ai/mcp/client) for details.

If you only need the model provider’s native MCP support without a local fallback, you can use [ MCPServerTool](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MCPServerTool) as a 

[native tool](/docs/ai/overview/native-tools#mcp-server-tool)directly.

For building MCP servers with Pydantic AI agents, see the [MCP server documentation](/docs/ai/mcp/server).

# Citations

1. Source page: https://pydantic.dev/docs/ai/mcp/overview
