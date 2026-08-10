---
type: Web Page
title: Tool Search | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/tool-search
timestamp: '2026-08-10T07:48:56.025339+00:00'
---

# Tool Search

The `ToolSearch`[capability](/docs/ai/capabilities/overview) handles model-driven discovery of searchable tools marked with `defer_loading=True`, so agents with large toolsets only pay tokens for the tools the model needs. Like the [provider-adaptive tools](/docs/ai/capabilities/overview#provider-adaptive-tools) above, it picks the best path for the active model — native server-executed search on Anthropic and OpenAI Responses, a local `search_tools` function tool elsewhere — and is auto-injected into every agent when searchable deferred tools exist. Bundle-level disclosure is covered by [on-demand capabilities](/docs/ai/capabilities/on-demand).

Pass an explicit [`ToolSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) to pick a specific [`strategy`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch.strategy) (`'keywords'`, `'bm25'`, `'regex'`, or a custom callable) or tune the local fallback:

When the local `search_tools` function tool is used, its retry budget follows the agent’s tool budget — so `Agent(retries={'tools': N})` gives the model `N` attempts to correct a malformed `queries` argument, on the same [precedence ladder](/docs/ai/tools-toolsets/tools-advanced#which-retry-limit-wins) as any other tool. A search that finds no matches returns normally and never spends a retry.

See [Tool Search](/docs/ai/tools-toolsets/tools-advanced#tool-search) for when to reach for it, the full strategy table, and provider support details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/tool-search
