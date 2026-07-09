---
type: Web Page
title: Server | Pydantic Docs
resource: https://pydantic.dev/docs/ai/mcp/server
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Server

Pydantic AI models can also be used within MCP Servers.

Here’s a simple example of a [Python MCP server](https://github.com/modelcontextprotocol/python-sdk) using Pydantic AI within a tool call:

This server can be queried with any MCP client. Here is an example using the Python SDK directly:

When Pydantic AI agents are used within MCP servers, they can use sampling via [ MCPSamplingModel](/docs/ai/api/models/mcp-sampling/#pydantic_ai.models.mcp_sampling.MCPSamplingModel).

We can extend the above example to use sampling so instead of connecting directly to the LLM, the agent calls back through the MCP client to make LLM calls.

The [above](#simple-client) client does not support sampling, so if you tried to use it with this server you’d get an error.

The simplest way to support sampling in an MCP client is to [use](/docs/ai/mcp/client#mcp-sampling) a Pydantic AI agent as the client, but if you wanted to support sampling with the vanilla MCP SDK, you could do so like this:

*(This example is complete, it can be run “as is”)*

# Citations

1. Source page: https://pydantic.dev/docs/ai/mcp/server
