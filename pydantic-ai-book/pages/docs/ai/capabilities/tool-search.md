---
type: Web Page
title: Tool Search | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/tool-search
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Tool Search

The `ToolSearch`[capability](/docs/ai/capabilities/overview) handles discovery of tools marked with `defer_loading=True`, so agents with large toolsets only pay tokens for the tools the model needs. Like the [provider-adaptive tools](/docs/ai/capabilities/overview#provider-adaptive-tools) above, it picks the best path for the active model — native server-executed search on Anthropic and OpenAI Responses, a local `search_tools` function tool elsewhere — and is auto-injected into every agent with zero overhead when no deferred tools exist.

Pass an explicit [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) to pick a specific 

[(](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch.strategy)

`strategy``'keywords'`, `'bm25'`, `'regex'`, or a custom callable) or tune the local fallback:See [Tool Search](/docs/ai/tools-toolsets/tools-advanced#tool-search) for when to reach for it, the full strategy table, and provider support details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/tool-search
