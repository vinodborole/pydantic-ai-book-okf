---
type: Web Page
title: OpenRouter | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/openrouter
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# OpenRouter

To use `OpenRouterModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openrouter` optional group:

To use [OpenRouter](https://openrouter.ai), first create an API key at [openrouter.ai/keys](https://openrouter.ai/keys).

You can set the `OPENROUTER_API_KEY` environment variable and use [`OpenRouterProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openrouter.OpenRouterProvider) by name:

```
from pydantic_ai import Agent
agent = Agent('openrouter:anthropic/claude-sonnet-4.6')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openrouter import OpenRouterModel
from pydantic_ai.providers.openrouter import OpenRouterProvider
model = OpenRouterModel(
    'anthropic/claude-sonnet-4.6',
    provider=OpenRouterProvider(api_key='your-openrouter-api-key'),
)
agent = Agent(model)
...
```
OpenRouter has an [app attribution](https://openrouter.ai/docs/app-attribution) feature to track your application in their public ranking and analytics.

You can pass in an `app_url` and `app_title` when initializing the provider to enable app attribution.

```
from pydantic_ai.providers.openrouter import OpenRouterProvider
provider=OpenRouterProvider(
    api_key='your-openrouter-api-key',
    app_url='https://your-app.com',
    app_title='Your App',
),
...
```
You can customize model behavior using [`OpenRouterModelSettings`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings):

```
from pydantic_ai import Agent
from pydantic_ai.models.openrouter import OpenRouterModel, OpenRouterModelSettings
settings = OpenRouterModelSettings(
    openrouter_reasoning={
        'effort': 'high',
    },
    openrouter_usage={
        'include': True,
    }
)
model = OpenRouterModel('openai/gpt-5.2')
agent = Agent(model, model_settings=settings)
...
```
For Anthropic models via OpenRouter, you can enable eager input streaming to reduce latency for tool calls with large inputs.
Set [`anthropic_eager_input_streaming`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_eager_input_streaming) in [`AnthropicModelSettings`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings):

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModelSettings
from pydantic_ai.models.openrouter import OpenRouterModel
model = OpenRouterModel('anthropic/claude-sonnet-4-5')
settings = AnthropicModelSettings(anthropic_eager_input_streaming=True)
agent = Agent(model, model_settings=settings)
...
```
Anthropic models don’t support a forced [`tool_choice`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.tool_choice) while [thinking](/docs/ai/capabilities/thinking) is enabled. Where the Anthropic API rejects that combination outright, OpenRouter silently drops the `reasoning` field from the request instead, so the response comes back with no thinking at all. With thinking enabled on an `anthropic/` model:

- An explicit `tool_choice='required'` (or a list of tool names) raises a[`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) ; disable thinking or use`tool_choice='auto'` .
- A `required` choice that Pydantic AI resolved on your behalf (e.g. from an[output tool](/docs/ai/core-concepts/output#tool-output) ) falls back softly to`'auto'` , so thinking is preserved. If the resolved choice named a single tool, the available tool list is filtered to that tool while`tool_choice` remains`'auto'` . The model may therefore answer with text instead of calling it; when an output tool is required, Pydantic AI retries with a prompt to call a tool.

OpenRouter supports [prompt caching](https://openrouter.ai/docs/guides/best-practices/prompt-caching) for downstream providers that implement it. Pydantic AI’s OpenRouter cache settings control explicit `cache_control` breakpoints for Anthropic and Gemini models:

1. **Cache System Instructions** : Set[`OpenRouterModelSettings.openrouter_cache_instructions`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings.openrouter_cache_instructions) to`True` or specify`'5m'` /`'1h'` directly
2. **Cache the Last Message** : Set[`OpenRouterModelSettings.openrouter_cache_messages`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings.openrouter_cache_messages) to`True` to automatically cache the last message in the conversation
3. **Cache Tool Definitions** : Set[`OpenRouterModelSettings.openrouter_cache_tool_definitions`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings.openrouter_cache_tool_definitions) to`True` or specify`'5m'` /`'1h'` directly
4. **Fine-Grained Control with [`CachePoint`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.CachePoint)** : Insert a`CachePoint` marker in user messages to cache everything before it

[`OpenRouterModel`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModel) does not currently translate [`CachePoint`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.CachePoint) into OpenAI’s breakpoint protocol (OpenAI models on OpenRouter still get automatic caching). For explicit GPT-5.6 breakpoints, combine [`OpenAIResponsesModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModel) (or [`OpenAIChatModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModel)) with [`OpenRouterProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openrouter.OpenRouterProvider):

```
from pydantic_ai import Agent, CachePoint
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
from pydantic_ai.providers.openrouter import OpenRouterProvider
model = OpenAIResponsesModel(
    'openai/gpt-5.6-sol',
    provider=OpenRouterProvider(api_key='your-openrouter-api-key'),
)
settings = OpenAIResponsesModelSettings(
    openai_prompt_cache_key='product-docs-v1',
    openai_prompt_cache_options={'mode': 'explicit', 'ttl': '30m'},
    # OpenRouter also offers Azure routes for GPT-5.6, where explicit caching is not documented.
    extra_body={'provider': {'only': ['openai']}},
)
agent = Agent(model, model_settings=settings)
result = agent.run_sync([
    'Long-lived reference material...',
    CachePoint(),
    'Answer using the reference material.',
])
```
The OpenRouter Responses API uses the same request-wide TTL and usage fields as OpenAI. Restricting the downstream provider to `openai` avoids routing explicit-cache requests to endpoints where these fields are not documented. OpenRouter currently documents explicit breakpoints only on text blocks, so place `CachePoint` markers after text content.

Use [`OpenRouterModelSettings`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings) to enable explicit caching for system instructions, the last conversation message, and tool definitions:

```
from pydantic_ai import Agent, RunContext
from pydantic_ai.models.openrouter import OpenRouterModel, OpenRouterModelSettings
model = OpenRouterModel('anthropic/claude-sonnet-4.6')
agent = Agent(
    model,
    instructions='You are a specialized assistant with deep domain knowledge...',
    model_settings=OpenRouterModelSettings(
        openrouter_cache_instructions=True,  # Cache system instructions (broadly supported)
        openrouter_cache_messages=True,  # Cache the last message (best with Anthropic)
        openrouter_cache_tool_definitions=True,  # Cache tool definitions (Anthropic only)
    ),
)
@agent.tool
def search_docs(ctx: RunContext, query: str) -> str:
    """Search documentation."""
    return f'Results for {query}'
...
```
Each setting accepts `True` or an explicit `'5m'` / `'1h'` TTL value. `True` sends Anthropic’s default `'5m'` TTL for Anthropic models; Gemini ignores TTL values and manages cache lifetime itself. Check `result.usage.cache_write_tokens` on initial writes and `result.usage.cache_read_tokens` on reuse, including subsequent calls with `message_history=result.all_messages()`.

OpenRouter uses [provider sticky routing](https://openrouter.ai/docs/guides/best-practices/prompt-caching#provider-sticky-routing) after prompt-cached requests to improve cache locality. For cache-sensitive workflows that need stricter provider control or disabled fallbacks, also set [`openrouter_provider`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModelSettings.openrouter_provider), for example with `{'order': ['anthropic'], 'allow_fallbacks': False}`.

Use [`CachePoint`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.CachePoint) markers to control exactly where cache boundaries are placed:

```
from pydantic_ai import Agent, CachePoint
from pydantic_ai.models.openrouter import OpenRouterModel
model = OpenRouterModel('anthropic/claude-sonnet-4.6')
agent = Agent(model)
prompt = [
    'Long reference document or context to cache...',
    CachePoint(),  # Cache everything before this point
    'Now answer my question about the context above',
]
...
```
Pass the prompt list to `agent.run_sync(prompt)`. Everything before the `CachePoint()` marker is cached. You can place multiple markers for fine-grained control over cache boundaries.

OpenRouter supports web search via its [plugins](https://openrouter.ai/docs/guides/features/plugins/web-search). You can enable it using the [`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool).

You can customize the web search behavior using the `search_context_size` parameter on [`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool):

```
from pydantic_ai import Agent
from pydantic_ai.capabilities import NativeTool
from pydantic_ai.models.openrouter import OpenRouterModel
from pydantic_ai.native_tools import WebSearchTool
tool = WebSearchTool(search_context_size='high')
model = OpenRouterModel('openai/gpt-4.1')
agent = Agent(
    model,
    capabilities=[NativeTool(tool)],
)
result = agent.run_sync('What is the latest news in AI?')
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/openrouter
