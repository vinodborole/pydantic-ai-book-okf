---
type: Web Page
title: X Search | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/x-search
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# X Search

The `XSearch`[capability](/docs/ai/capabilities/overview/) gives your agent search over X (Twitter) posts. It’s a [provider-adaptive tool](/docs/ai/capabilities/overview/#provider-adaptive-tools) backed by [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool) on the native side — see [X Search Tool](/docs/ai/tools-toolsets/native-tools/#x-search-tool) for configuration options.

Unlike [Web Search](/docs/ai/capabilities/web-search/) and [Web Fetch](/docs/ai/capabilities/web-fetch/), there is no default non-xAI fallback: X search is only available natively on xAI models. If your agent is not running on an xAI model, set `fallback_model` explicitly to an xAI model that supports [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool), and search requests are delegated to that model as a subagent tool:

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/x-search
