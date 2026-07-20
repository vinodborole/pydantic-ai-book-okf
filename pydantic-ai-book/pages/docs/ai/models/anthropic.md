---
type: Web Page
title: Anthropic | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/anthropic
timestamp: '2026-07-20T09:23:04.251034+00:00'
---

# Anthropic

To use `AnthropicModel` models, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `anthropic` optional group:

To use [Anthropic](https://anthropic.com) through their API, go to [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) to generate an API key.

`AnthropicModelName` contains a list of available Anthropic models.

Once you have the API key, you can set it as an environment variable:

You can then use `AnthropicModel` by name:

```
from pydantic_ai import Agent
agent = Agent('anthropic:claude-sonnet-4-6')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
model = AnthropicModel('claude-sonnet-4-5')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
from pydantic_ai.providers.anthropic import AnthropicProvider
model = AnthropicModel(
    'claude-sonnet-4-5', provider=AnthropicProvider(api_key='your-api-key')
)
agent = Agent(model)
...
```
You can customize the `AnthropicProvider` with a custom `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
from pydantic_ai.providers.anthropic import AnthropicProvider
custom_http_client = AsyncClient(timeout=30)
model = AnthropicModel(
    'claude-sonnet-4-5',
    provider=AnthropicProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```
You can customize model behavior using [ AnthropicModelSettings](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings):

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel, AnthropicModelSettings
model = AnthropicModel('claude-sonnet-4-5')
settings = AnthropicModelSettings(
    temperature=0.2,
    top_k=40,
    service_tier='auto',
)
agent = Agent(model, model_settings=settings)
...
```
Anthropic supports controlling the [service tier](https://platform.claude.com/docs/en/api/service-tiers) to manage latency and throughput.
You can use the unified [ service_tier](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.service_tier) field or the provider-specific 

[field.](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_service_tier)

`anthropic_service_tier``anthropic_service_tier` takes precedence over the unified field when both are set, and accepts Anthropic’s native values (`'auto'` or `'standard_only'`).The unified field maps as follows for Anthropic:

- `'auto'`: passed through as- `'auto'`(Anthropic’s native value — uses priority capacity when available).
- `'default'`: maps to- `'standard_only'`(forces the standard tier, opting out of priority capacity).
- `'flex'`and- `'priority'`are not part of Anthropic’s tier model and are silently ignored.

You can use Anthropic models through cloud platforms by passing a custom client to [ AnthropicProvider](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.anthropic.AnthropicProvider).

To use Claude models via [AWS Bedrock](https://aws.amazon.com/bedrock/claude/), follow the [Anthropic documentation](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock) on how to set up a Bedrock client and then pass it to `AnthropicProvider`. Both the newer `AsyncAnthropicBedrockMantle` client (recommended by Anthropic, using the Messages API) and the legacy `AsyncAnthropicBedrock` client (using the `InvokeModel` API with ARN-versioned model IDs) are supported:

```
from anthropic import AsyncAnthropicBedrockMantle
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
from pydantic_ai.providers.anthropic import AnthropicProvider
bedrock_client = AsyncAnthropicBedrockMantle()  # Uses AWS credentials from environment
provider = AnthropicProvider(anthropic_client=bedrock_client)
model = AnthropicModel('anthropic.claude-haiku-4-5', provider=provider)
agent = Agent(model)
...
```
To use Claude models via [Google Cloud Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/partner-models/use-claude), follow the [Anthropic documentation](https://docs.anthropic.com/en/api/claude-on-vertex-ai) on how to set up an `AsyncAnthropicVertex` client and then pass it to `AnthropicProvider`:

```
from anthropic import AsyncAnthropicVertex
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
from pydantic_ai.providers.anthropic import AnthropicProvider
vertex_client = AsyncAnthropicVertex(region='us-east5', project_id='your-project-id')
provider = AnthropicProvider(anthropic_client=vertex_client)
model = AnthropicModel('claude-sonnet-4-5', provider=provider)
agent = Agent(model)
...
```
To use Claude models via [Microsoft Foundry](https://ai.azure.com/), follow the [Anthropic documentation](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry) on how to set up an `AsyncAnthropicFoundry` client and then pass it to `AnthropicProvider`:

```
from anthropic import AsyncAnthropicFoundry
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel
from pydantic_ai.providers.anthropic import AnthropicProvider
foundry_client = AsyncAnthropicFoundry(
    api_key='your-foundry-api-key',  # Or set ANTHROPIC_FOUNDRY_API_KEY
    resource='your-resource-name',
)
provider = AnthropicProvider(anthropic_client=foundry_client)
model = AnthropicModel('claude-sonnet-4-5', provider=provider)
agent = Agent(model)
...
```
See [Anthropic’s Microsoft Foundry documentation](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry) for setup instructions including Entra ID authentication.

Anthropic’s [task budgets](https://platform.claude.com/docs/en/build-with-claude/task-budgets) let you give Claude an advisory token budget for a full agentic loop — including thinking, tool calls, tool results, and output — so the model can pace itself and finish gracefully as the budget is consumed. Configure them with [ AnthropicModelSettings.anthropic_task_budget](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_task_budget), which takes an 

[payload and maps to](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicTaskBudget)

`AnthropicTaskBudget``output_config.task_budget`.Pydantic AI automatically enables Anthropic’s required `task-budgets-2026-03-13` beta when this setting is present. Support is currently limited to native Anthropic `claude-opus-4-7`, `claude-opus-4-8`, and `claude-sonnet-5` requests, not Bedrock, Vertex, or Microsoft Foundry Anthropic model IDs.

Task budgets compose with [ anthropic_effort](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_effort): effort tunes per-step reasoning depth, while task budgets cap total work across the loop. Both fields end up under the same 

`output_config` object.If you use [ AnthropicCompaction](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicCompaction) for server-side compaction, you can skip this section: the server tracks the countdown itself, so leave 

`remaining` unset and let `total` self-regulate.The `remaining` field on `task_budget` is for *client-side* compaction patterns where you summarize earlier turns yourself between requests, so the server has no memory of how much budget was spent before the rewrite. Pydantic AI does not track `remaining` for you — accumulate token usage across requests yourself (e.g. from [ RunUsage](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage) on each run) and pass the updated value on the next request so the countdown continues from where you left off rather than resetting to 

`total`. Setting `remaining` also invalidates any prompt-cache prefix that contains the budget, so if you want to preserve caching, set `total` once and let the server self-regulate against the running countdown.Anthropic supports [prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) to reduce costs by caching parts of your prompts. Pydantic AI supports automatic caching, per-block message caching, and explicit cache breakpoints:

The simplest way to enable prompt caching is with [ AnthropicModelSettings.anthropic_cache](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_cache). This uses Anthropic’s 

[automatic caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching#automatic-caching), passing a top-level

`cache_control` parameter so the server automatically applies a cache breakpoint to the last cacheable block in each request:```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='You are a helpful assistant.',
    model_settings=AnthropicModelSettings(
        anthropic_cache=True,
    ),
)
result1 = agent.run_sync('What is the capital of France?')
result2 = agent.run_sync(
    'What is the capital of Germany?', message_history=result1.all_messages()
)
print(f'Cache write: {result1.usage.cache_write_tokens}')
print(f'Cache read: {result2.usage.cache_read_tokens}')
print(f'Cache hit ratio: {result2.usage.cache_hit_ratio}')
```
This is ideal for multi-turn conversations where the cache breakpoint should move forward as the conversation grows. You can also specify a custom TTL with `anthropic_cache='1h'`.

As an alternative to `anthropic_cache`, [ AnthropicModelSettings.anthropic_cache_messages](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_cache_messages) adds per-block 

`cache_control` to the last content block of the final message instead of using Anthropic’s top-level automatic caching parameter. Use this with Anthropic-compatible gateways and proxies (such as MiniMax, OpenRouter, or LiteLLM) that accept the Anthropic message format but don’t support top-level automatic caching:```
from anthropic import AsyncAnthropic
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModel, AnthropicModelSettings
from pydantic_ai.providers.anthropic import AnthropicProvider
client = AsyncAnthropic(
    api_key='your-api-key',
    base_url='https://your-anthropic-compatible-gateway.example.com',
)
model = AnthropicModel(
    'claude-sonnet-4-6',
    provider=AnthropicProvider(anthropic_client=client),
)
agent = Agent(
    model,
    model_settings=AnthropicModelSettings(
        anthropic_cache_messages=True,
    ),
)
result = agent.run_sync('What is the capital of France?')
print(result.output)
```
You can also specify a custom TTL with `anthropic_cache_messages='1h'`. `anthropic_cache_messages` cannot be combined with `anthropic_cache`.

In addition to automatic caching, Pydantic AI provides several ways to place cache breakpoints on specific content:

- **Cache User Messages with**: Insert a- `CachePoint`- `CachePoint`marker in your user messages to cache everything before it
- **Cache the Final Message Block**: Set- `AnthropicModelSettings.anthropic_cache_messages`- `True`(uses 5m TTL by default) or specify- `'5m'`/- `'1h'`directly
- **Cache System Instructions**: Set- `AnthropicModelSettings.anthropic_cache_instructions`- `True`(uses 5m TTL by default) or specify- `'5m'`/- `'1h'`directly
- **Cache Tool Definitions**: Set- `AnthropicModelSettings.anthropic_cache_tool_definitions`- `True`(uses 5m TTL by default) or specify- `'5m'`/- `'1h'`directly

Combine automatic caching with explicit breakpoints for maximum savings. Automatic caching handles the conversation, while explicit breakpoints pin system instructions and tool definitions:

```
from pydantic_ai import Agent, RunContext
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='Detailed instructions...',
    model_settings=AnthropicModelSettings(
        anthropic_cache=True,                   # Server auto-caches last block
        anthropic_cache_instructions=True,      # Explicitly cache system instructions
        anthropic_cache_tool_definitions='1h',  # Explicitly cache tool definitions with 1h TTL
    ),
)
@agent.tool
def search_docs(ctx: RunContext, query: str) -> str:
    """Search documentation."""
    return f'Results for {query}'
result = agent.run_sync('Search for Python best practices')
print(result.output)
```
When you use `anthropic_cache_instructions` with both static and dynamic [instructions](/docs/ai/core-concepts/agent#instructions), Pydantic AI automatically places the cache boundary at the optimal point. Static instructions (from `Agent(instructions=...)`) are sorted before dynamic instructions (from `@agent.instructions` functions or [toolsets](/docs/ai/tools-toolsets/toolsets)), and the cache point is placed after the last static instruction block.

This means your stable, static instructions are cached efficiently, while dynamic instructions (which may change between requests) remain outside the cache boundary and don’t cause cache invalidation.

Static instructions are cached across requests.

Enables smart cache placement at the static/dynamic boundary.

Dynamic instructions change per-request and are not cached.

Use manual `CachePoint` markers to control cache locations precisely:

```
from pydantic_ai import Agent, CachePoint
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='Instructions...',
)
# Manually control cache points for specific content blocks
result = agent.run_sync([
    'Long context from documentation...',
    CachePoint(),  # Cache everything up to this point
    'First question'
])
print(result.output)
```
Access cache usage statistics via `result.usage`:

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='Instructions...',
    model_settings=AnthropicModelSettings(
        anthropic_cache=True,
    ),
)
result = agent.run_sync('Your question')
usage = result.usage
print(f'Cache write tokens: {usage.cache_write_tokens}')
print(f'Cache read tokens: {usage.cache_read_tokens}')
```
Anthropic enforces a maximum of 4 cache points per request. Pydantic AI automatically manages this limit to ensure your requests always comply without errors.

Cache points can come from several sources:

- **Automatic caching**: Via- `anthropic_cache`(the server applies 1 cache point to the last cacheable block)
- **Final message block**: Via- `anthropic_cache_messages`setting (adds cache point to last message content block)
- **System Prompt**: Via- `anthropic_cache_instructions`setting (adds cache point to last system prompt block)
- **Tool Definitions**: Via- `anthropic_cache_tool_definitions`setting (adds cache point to last tool definition)
- **Messages**: Via- `CachePoint`markers (adds cache points to message content)

Each setting uses **at most 1 cache point**, but you can combine them — except `anthropic_cache` and `anthropic_cache_messages`, which are mutually exclusive. If the total exceeds 4, Pydantic AI automatically trims excess cache points from older messages.

Define an agent with automatic caching plus explicit breakpoints:

```
from pydantic_ai import Agent, CachePoint
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='Detailed instructions...',
    model_settings=AnthropicModelSettings(
        anthropic_cache=True,                   # 1 cache point (server-applied)
        anthropic_cache_instructions=True,      # 1 cache point
        anthropic_cache_tool_definitions=True,  # 1 cache point
    ),
)
@agent.tool_plain
def my_tool() -> str:
    return 'result'
# 3 of 4 slots used (1 automatic + 1 instructions + 1 tools)
# Room for 1 more explicit CachePoint marker
result = agent.run_sync([
    'Context', CachePoint(),  # 4th cache point - OK
    'Question'
])
print(result.output)
usage = result.usage
print(f'Cache write tokens: {usage.cache_write_tokens}')
print(f'Cache read tokens: {usage.cache_read_tokens}')
```
When explicit cache points from all sources (settings + `CachePoint` markers) exceed the available budget, Pydantic AI automatically removes excess cache points from **older message content** (keeping the most recent ones).

Define an agent with 2 explicit cache points from settings:

```
from pydantic_ai import Agent, CachePoint
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    instructions='Instructions...',
    model_settings=AnthropicModelSettings(
        anthropic_cache_instructions=True,      # 1 cache point
        anthropic_cache_tool_definitions=True,  # 1 cache point
    ),
)
@agent.tool_plain
def search() -> str:
    return 'data'
# Already using 2 cache points (instructions + tools)
# Can add 2 more CachePoint markers (4 total limit)
result = agent.run_sync([
    'Context 1', CachePoint(),  # Oldest - will be removed
    'Context 2', CachePoint(),  # Will be kept (3rd point)
    'Context 3', CachePoint(),  # Will be kept (4th point)
    'Question'
])
# Final cache points: instructions + tools + Context 2 + Context 3 = 4
print(result.output)
usage = result.usage
print(f'Cache write tokens: {usage.cache_write_tokens}')
print(f'Cache read tokens: {usage.cache_read_tokens}')
```
**Key Points**:

- System and tool cache points are **always preserved**
- `anthropic_cache`counts as 1 cache point, just like- `anthropic_cache_instructions`and- `anthropic_cache_tool_definitions`
- Excess `CachePoint`markers in messages are removed from oldest to newest when the limit is exceeded
- This ensures critical caching (instructions/tools) is maintained while still benefiting from message-level caching

Fast mode provides higher output tokens per second and is currently supported on **Claude Opus 4.6**, **Claude Opus 4.7**, and **Claude Opus 4.8**. It is a research preview. Set [ anthropic_speed](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_speed) to 

`'fast'` to enable it; Pydantic AI automatically adds the required `fast-mode-2026-02-01` beta. On unsupported models, `anthropic_speed='fast'` is ignored with a `UserWarning`. For pricing, rate limits, and the latest list of supported models, see the [Anthropic fast mode docs](https://platform.claude.com/docs/en/build-with-claude/fast-mode).

```
from pydantic_ai import Agent
from pydantic_ai.models.anthropic import AnthropicModelSettings
agent = Agent(
    'anthropic:claude-opus-4-8',
    model_settings=AnthropicModelSettings(anthropic_speed='fast'),
)
...
```
Most Anthropic models let you force a tool call via [ tool_choice='required'](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.tool_choice) (or a list of tool names), except while thinking is enabled. 

**Claude Fable 5**and the

**Claude Mythos**models reject a forced tool choice unconditionally — even without thinking — so Pydantic AI marks them with

[.](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.anthropic.AnthropicModelProfile.anthropic_supports_forced_tool_choice)

`anthropic_supports_forced_tool_choice=False`On a model that doesn’t support forcing:

- An explicit `tool_choice='required'`(or a list of tool names) raises a`UserError``tool_choice='auto'`instead.
- A `required`choice that Pydantic AI resolved on your behalf (e.g. from an[output tool](/docs/ai/core-concepts/output#tool-output)) falls back softly to`'auto'`, with the available tools filtered to the requested set so the model can still only pick from them. Filtering the tool definitions invalidates Anthropic’s prompt cache, since the cached prefix includes the tool array.

Anthropic supports [automatic context compaction](https://docs.anthropic.com/en/docs/build-with-claude/compaction) to manage long conversations. When input tokens exceed a configured threshold, the API automatically generates a summary that replaces older messages while preserving context.

The easiest way to enable compaction is with the [ AnthropicCompaction](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicCompaction) capability:

The capability accepts:

- `token_threshold`
- `instructions`
- `pause_after_compaction`- `True`, the response stops after the compaction block with- `stop_reason='compaction'`, allowing explicit handling before continuing.

Alternatively, you can configure compaction directly via model settings using [ anthropic_context_management](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_context_management):

By default, Pydantic AI chooses a compatible Anthropic code execution tool version for the selected model. You can override this with [ AnthropicModelSettings.anthropic_code_execution_tool_version](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_code_execution_tool_version) when you need a specific supported Anthropic tool version:

Pydantic AI raises a [ UserError](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) if you explicitly select a tool version that the model does not support.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/anthropic
