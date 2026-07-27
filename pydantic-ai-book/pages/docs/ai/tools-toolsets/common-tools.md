---
type: Web Page
title: Common Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/common-tools
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Common Tools

Pydantic AI ships with native tools that can be used to enhance your agent’s capabilities.

The DuckDuckGo search tool allows you to search the web for information. It is built on top of the
[DuckDuckGo API](https://github.com/deedy5/ddgs).

To use [ duckduckgo_search_tool](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.duckduckgo.duckduckgo_search_tool), you need to install

[with the](/docs/ai/overview/install#slim-install)

`pydantic-ai-slim``duckduckgo` optional group:Here’s an example of how you can use the DuckDuckGo search tool with an agent:

The web fetch tool allows your agent to fetch the content of web pages and convert them to markdown.
It uses [SSRF protection](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery) to prevent server-side request forgery attacks.

To use [ web_fetch_tool](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.web_fetch.web_fetch_tool), you need to install

[with the](/docs/ai/overview/install#slim-install)

`pydantic-ai-slim``web-fetch` optional group:Here’s an example of how you can use the web fetch tool with an agent:

The Tavily search tool allows you to search the web for information. It is built on top of the [Tavily API](https://tavily.com/).

To use [ tavily_search_tool](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.tavily.tavily_search_tool), you need to install

[with the](/docs/ai/overview/install#slim-install)

`pydantic-ai-slim``tavily` optional group:Here’s an example of how you can use the Tavily search tool with an agent:

The `tavily_search_tool` factory accepts optional parameters that control search behavior. `max_results` is always developer-controlled and never appears in the LLM tool schema. Other parameters, when provided, are fixed for all searches and hidden from the LLM’s tool schema. Parameters left unset remain available for the LLM to set per-call.

For example, you can lock in `max_results` and `include_domains` at tool creation time while still letting the LLM control `exclude_domains`:

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/common-tools
