---
type: Web Page
title: Common Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/common-tools
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Common Tools

Pydantic AI ships with native tools that can be used to enhance your agent’s capabilities.

The DuckDuckGo search tool allows you to search the web for information. It is built on top of the DuckDuckGo API.

To use `duckduckgo_search_tool`, you need to install
`pydantic-ai-slim` with the `duckduckgo` optional group:

Here’s an example of how you can use the DuckDuckGo search tool with an agent:

The web fetch tool allows your agent to fetch the content of web pages and convert them to markdown. It uses SSRF protection to prevent server-side request forgery attacks.

To use `web_fetch_tool`, you need to install
`pydantic-ai-slim` with the `web-fetch` optional group:

Here’s an example of how you can use the web fetch tool with an agent:

The Tavily search tool allows you to search the web for information. It is built on top of the Tavily API.

To use `tavily_search_tool`, you need to install
`pydantic-ai-slim` with the `tavily` optional group:

Here’s an example of how you can use the Tavily search tool with an agent:

The `tavily_search_tool` factory accepts optional parameters that control search behavior. `max_results` is always developer-controlled and never appears in the LLM tool schema. Other parameters, when provided, are fixed for all searches and hidden from the LLM’s tool schema. Parameters left unset remain available for the LLM to set per-call.

For example, you can lock in `max_results` and `include_domains` at tool creation time while still letting the LLM control `exclude_domains`:

Exa is a neural search engine that finds high-quality, relevant results across billions of web pages. It provides several tools including web search, finding similar pages, content retrieval, and AI-powered answers.

To use Exa tools, you need to install `pydantic-ai-slim` with the `exa` optional group:

You can use Exa tools individually or as a toolset. The following tools are available:

- `exa_search_tool`: Search the web with various search types (auto, keyword, neural, fast, deep)
- `exa_find_similar_tool`: Find pages similar to a given URL
- `exa_get_contents_tool`: Get full text content from URLs
- `exa_answer_tool`: Get AI-powered answers with citations

For better efficiency when using multiple Exa tools, use `ExaToolset`
which shares a single API client across all tools. You can configure which tools to include:

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/common-tools
