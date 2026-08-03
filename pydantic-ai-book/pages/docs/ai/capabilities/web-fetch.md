---
type: Web Page
title: Web Fetch | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/web-fetch
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Web Fetch

The `WebFetch`[capability](/docs/ai/capabilities/overview) lets your agent fetch the contents of URLs. Like all [provider-adaptive tools](/docs/ai/capabilities/overview#provider-adaptive-tools), it prefers the provider’s native web fetch tool and can fall back to a local implementation on other models.

[`WebFetch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch) defaults to native-only. Backed by [`WebFetchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebFetchTool) on the native side (see [Web Fetch Tool](/docs/ai/tools-toolsets/native-tools#web-fetch-tool) for provider support and configuration) — pass `native=WebFetchTool(...)` directly for full control.

For the local side, pass `local=True` for the bundled [markdownify-based fetch tool](/docs/ai/tools-toolsets/common-tools#web-fetch-tool) (requires the `web-fetch` optional group), or any callable, [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset).

Native constraint fields: `allowed_domains`, `blocked_domains`, `max_uses`, `enable_citations`, `max_content_tokens`. Only `max_uses` requires native; domain filters are enforced locally when native isn’t available.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/web-fetch
