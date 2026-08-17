---
type: Web Page
title: Web Search | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/web-search
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Web Search

The `WebSearch`[capability](/docs/ai/capabilities/overview/) gives your agent web search. Like all [provider-adaptive tools](/docs/ai/capabilities/overview/#provider-adaptive-tools), it uses the provider’s native web search when the model supports it and can fall back to a local implementation on other models.

[`WebSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch) defaults to native-only. Backed by [`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool) on the native side (see [Web Search Tool](/docs/ai/tools-toolsets/native-tools/#web-search-tool) for provider support and configuration) — pass `native=WebSearchTool(...)` directly when you need full control over the native instance.

For the local side, pass `local='duckduckgo'` (or `local=True`) for a [DuckDuckGo](/docs/ai/tools-toolsets/common-tools/#duckduckgo-search-tool) fallback (requires the `duckduckgo` optional group); for other search providers, use a [Tavily](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.tavily.tavily_search_tool) wrapper from [`common_tools`](/docs/ai/tools-toolsets/common-tools/), the [`ExaSearchToolset`](https://pydantic.dev/docs/ai/harness/exa-search/) from the Pydantic AI Harness, or any callable, [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset).

Native configuration fields: `search_context_size`, `user_location`, `blocked_domains`, `allowed_domains`,
`max_uses`, and OpenAI Responses’ `external_web_access`. The domain and `max_uses` constraints require native
support. Setting `external_web_access=False` also requires native support because a local fallback cannot guarantee
cached or indexed-only search.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/web-search
