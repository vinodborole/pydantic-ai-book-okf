---
type: Web Page
title: X Search | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/x-search
timestamp: '2026-09-07T12:01:58.556264+00:00'
---

# X Search

The `XSearch`[capability](/docs/ai/capabilities/overview/) gives your agent search over X (Twitter) posts. It’s a [provider-adaptive tool](/docs/ai/capabilities/overview/#provider-adaptive-tools) backed by [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool) on the native side — see [X Search Tool](/docs/ai/tools-toolsets/native-tools/#x-search-tool) for configuration options.

Unlike [Web Search](/docs/ai/capabilities/web-search/) and [Web Fetch](/docs/ai/capabilities/web-fetch/), there is no default non-xAI fallback: X search is only available natively on xAI models. If your agent is not running on an xAI model, set `fallback_subagent_model` explicitly to an xAI model that supports [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool), and search requests are delegated to that model as a subagent tool:

`native=` can be a factory: a callable taking [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) that returns [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool) or `None` — [`XSearchNativeTool`](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.x_search.XSearchNativeTool) is the type the `fallback_subagent_model` subagent accepts. It resolves on each model request, and again when the subagent runs — so keep it free of one-shot side effects. Both resolutions belong to the same run and carry the same `deps`, but they do not share a [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext): the subagent resolves from its own tool call, so `tool_call_id` and `tool_name` name that call instead of being `None`, and `messages` holds the run so far. Read `ctx.deps` for configuration that has to match across both. On the subagent, capability-level fields such as `include_output` override the factory result. See [Dynamic Configuration](/docs/ai/tools-toolsets/native-tools/#dynamic-configuration).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/x-search
