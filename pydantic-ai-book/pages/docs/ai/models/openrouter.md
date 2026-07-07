---
type: Web Page
title: OpenRouter | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/openrouter
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# OpenRouter

To use `OpenRouterModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openrouter` optional group:

To use OpenRouter, first create an API key at openrouter.ai/keys.

You can set the `OPENROUTER_API_KEY` environment variable and use `OpenRouterProvider` by name:

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
OpenRouter has an app attribution feature to track your application in their public ranking and analytics.

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
You can customize model behavior using `OpenRouterModelSettings`:

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
Set `anthropic_eager_input_streaming` in `AnthropicModelSettings`:

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModelSettings
from pydantic_ai.models.openrouter import OpenRouterModel
model = OpenRouterModel('anthropic/claude-sonnet-4-5')
settings = AnthropicModelSettings(anthropic_eager_input_streaming=True)
agent = Agent(model, model_settings=settings)
...
```
OpenRouter supports prompt caching for downstream providers that implement it. Pydantic AI’s OpenRouter cache settings control explicit `cache_control` breakpoints for Anthropic and Gemini models:

- **Cache System Instructions**: Set- `OpenRouterModelSettings.openrouter_cache_instructions`to- `True`or specify- `'5m'`/- `'1h'`directly
- **Cache the Last Message**: Set- `OpenRouterModelSettings.openrouter_cache_messages`to- `True`to automatically cache the last message in the conversation
- **Cache Tool Definitions**: Set- `OpenRouterModelSettings.openrouter_cache_tool_definitions`to- `True`or specify- `'5m'`/- `'1h'`directly
- **Fine-Grained Control with**: Insert a- `CachePoint`- `CachePoint`marker in user messages to cache everything before it

Use `OpenRouterModelSettings` to enable explicit caching for system instructions, the last conversation message, and tool definitions:

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

OpenRouter uses provider sticky routing after prompt-cached requests to improve cache locality. For cache-sensitive workflows that need stricter provider control or disabled fallbacks, also set `openrouter_provider`, for example with `{'order': ['anthropic'], 'allow_fallbacks': False}`.

Use `CachePoint` markers to control exactly where cache boundaries are placed:

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

OpenRouter supports web search via its plugins. You can enable it using the `WebSearchTool`.

You can customize the web search behavior using the `search_context_size` parameter on `WebSearchTool`:

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
