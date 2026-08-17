---
type: Web Page
title: MCP | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/mcp
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# MCP

[`MCP`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP) is a [provider-adaptive capability](/docs/ai/capabilities/overview/#provider-adaptive-tools) and the primary entry point for [MCP](/docs/ai/mcp/overview/) in Pydantic AI. It runs the MCP server locally by default — keeping credentials, hooks, and tracing under your control — and supports both URL-based servers and direct client / toolset / transport inputs.

Backed by [`MCPServerTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MCPServerTool) on the native side (see [MCP Server Tool](/docs/ai/tools-toolsets/native-tools/#mcp-server-tool) for provider support and configuration) — pass `native=MCPServerTool(...)` directly when you need full control (e.g. a different `id`, `authorization_token`, or `description` than the capability would derive). On the local side, `local=` accepts any [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) input (URL, `fastmcp.Client`, transport, in-process `FastMCP` server, script path, …) — non-toolset inputs are wrapped in `MCPToolset` automatically.

For lower-level access — managing the [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) lifecycle directly, advanced transport / client configuration, or using MCP servers without going through a capability — see the [MCP documentation](/docs/ai/mcp/overview/).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/mcp
