---
type: Web Page
title: Bedrock | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/bedrock
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Bedrock

[Amazon Bedrock](https://aws.amazon.com/bedrock/) exposes foundation models from many providers, and Pydantic AI reaches it through two separate AWS APIs. Pick the route by model prefix:

- **[Bedrock Converse](#bedrock-converse)** (`bedrock:` ) — the broadest catalog, including Anthropic, Amazon, Cohere, Meta, Mistral, DeepSeek, Qwen, and[many more](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelName) , through the Bedrock Runtime Converse API. This is the route for almost every Bedrock model.
- **[Bedrock Mantle](#bedrock-mantle)** (`bedrock-mantle:` ) — the modern OpenAI models (GPT-5.x and GPT-OSS), which Bedrock serves only through[Mantle](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html) ’s OpenAI-compatible API.

Both routes authenticate with the same AWS credentials. The `bedrock:` prefix always uses Converse; requesting a frontier OpenAI model (GPT-5.4 or newer) through it raises an error pointing you to `bedrock-mantle:`, since Converse doesn’t serve those models.

| Route | Prefix | Models | Optional group | Model class | 
|---|---|---|---|---|
| [Converse](#bedrock-converse) | `bedrock:` | Anthropic, Amazon, Cohere, Meta, Mistral, and [more](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelName) | `bedrock` | [`BedrockConverseModel`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockConverseModel) | 
| [Mantle](#bedrock-mantle) | `bedrock-mantle:` | OpenAI GPT-5.x and GPT-OSS | `bedrock-mantle` | [`BedrockMantleResponsesModel`](/docs/ai/api/models/bedrock_mantle/#pydantic_ai.models.bedrock_mantle.BedrockMantleResponsesModel) ,[`BedrockMantleChatModel`](/docs/ai/api/models/bedrock_mantle/#pydantic_ai.models.bedrock_mantle.BedrockMantleChatModel) | 

[`BedrockConverseModel`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockConverseModel) talks to the [Bedrock Runtime Converse API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html), which serves the broadest set of Bedrock models.

To use `BedrockConverseModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `bedrock` optional group:

To use [AWS Bedrock](https://aws.amazon.com/bedrock/), you’ll need an AWS account with Bedrock enabled and appropriate credentials. You can use either AWS credentials directly or a pre-configured boto3 client.

[`BedrockModelName`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelName) contains a list of available Bedrock models, including models from Anthropic, Amazon, Cohere, Meta, and Mistral.

You can set your AWS credentials as environment variables ([among other options](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html#using-environment-variables)):

You can then use `BedrockConverseModel` by name:

```
from pydantic_ai import Agent
agent = Agent('bedrock:anthropic.claude-sonnet-4-5-20250929-v1:0')
...
```
Or initialize the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockConverseModel
model = BedrockConverseModel('anthropic.claude-sonnet-4-5-20250929-v1:0')
agent = Agent(model)
...
```
You can customize the Bedrock Runtime API calls by adding additional parameters, such as [guardrail
configurations](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) and [performance settings](https://docs.aws.amazon.com/bedrock/latest/userguide/latency-optimized-inference.html). For a complete list of configurable parameters, refer to the
documentation for [`BedrockModelSettings`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings).

Bedrock supports controlling the [service tier](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html) to manage throughput and cost.
You can use the unified [`service_tier`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.service_tier) field or the provider-specific [`bedrock_service_tier`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_service_tier) field. `bedrock_service_tier` takes precedence over the unified field when both are set.

The unified field maps as follows for Bedrock:

- `'auto'` : the`serviceTier` field is omitted from the request, so AWS applies its server-side default (Standard tier).
- `'default'` : explicitly sent as`{'type': 'default'}` — opts out of any future server-side auto-promotion to premium tiers.
- `'flex'` : sent as`{'type': 'flex'}` .
- `'priority'` : sent as`{'type': 'priority'}` .

To request Bedrock’s `'reserved'` tier (which requires a pre-purchased capacity reservation), set [`bedrock_service_tier`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_service_tier) directly — it isn’t reachable through the unified field.

Bedrock supports [prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) on Anthropic models so you can reuse expensive context across requests. Pydantic AI provides four ways to use prompt caching:

1. **Cache User Messages with [`CachePoint`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.CachePoint)** : Insert a`CachePoint` marker to cache everything before it in the current user message. Pass`CachePoint(ttl='1h')` to opt into the extended cache duration.
2. **Cache System Instructions** : Set[`BedrockModelSettings.bedrock_cache_instructions`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_cache_instructions) to`True` (uses 5m TTL by default) or specify`'5m'` /`'1h'` directly. When you have both static and dynamic[instructions](/docs/ai/core-concepts/agent#instructions) , the cache point is placed after the last static instruction, so dynamic instructions can change without invalidating the static cache.
3. **Cache Tool Definitions** : Set[`BedrockModelSettings.bedrock_cache_tool_definitions`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_cache_tool_definitions) to`True` (uses 5m TTL by default) or specify`'5m'` /`'1h'` directly.
4. **Cache All Messages** : Set[`BedrockModelSettings.bedrock_cache_messages`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_cache_messages) to`True` (uses 5m TTL by default) or specify`'5m'` /`'1h'` directly to automatically cache the last user message.

Use `bedrock_cache_messages` to automatically cache the last user message:

```
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockModelSettings
agent = Agent(
    'bedrock:us.anthropic.claude-sonnet-4-5-20250929-v1:0',
    system_prompt='You are a helpful assistant.',
    model_settings=BedrockModelSettings(
        bedrock_cache_messages=True,  # Automatically caches the last message
    ),
)
# The last message is automatically cached - no need for manual CachePoint
result1 = agent.run_sync('What is the capital of France?')
# Subsequent calls with similar conversation benefit from cache
result2 = agent.run_sync('What is the capital of Germany?')
print(f'Cache write: {result1.usage.cache_write_tokens}')
print(f'Cache read: {result2.usage.cache_read_tokens}')
```
Combine multiple cache settings for maximum savings:

```
from pydantic_ai import Agent, RunContext
from pydantic_ai.models.bedrock import BedrockConverseModel, BedrockModelSettings
model = BedrockConverseModel('us.anthropic.claude-sonnet-4-5-20250929-v1:0')
agent = Agent(
    model,
    system_prompt='Detailed instructions...',
    model_settings=BedrockModelSettings(
        bedrock_cache_instructions=True,       # Cache system instructions
        bedrock_cache_tool_definitions='1h',   # Cache tool definitions with 1h TTL
        bedrock_cache_messages=True,           # Also cache the last message
    ),
)
@agent.tool
def search_docs(ctx: RunContext, query: str) -> str:
    """Search documentation."""
    return f'Results for {query}'
result = agent.run_sync('Search for Python best practices')
print(result.output)
```
Use manual `CachePoint` markers to control cache locations precisely:

```
from pydantic_ai import Agent, CachePoint
agent = Agent(
    'bedrock:us.anthropic.claude-sonnet-4-5-20250929-v1:0',
    system_prompt='Instructions...',
)
# Manually control cache points for specific content blocks
result = agent.run_sync([
    'Long context from documentation...',
    CachePoint(),  # Cache everything up to this point
    'First question'
])
print(result.output)
```
Access cache usage statistics via [`RequestUsage`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RequestUsage):

```
from pydantic_ai import Agent, CachePoint
agent = Agent('bedrock:us.anthropic.claude-sonnet-4-5-20250929-v1:0')
async def main():
    result = await agent.run(
        [
            'Reference material...',
            CachePoint(),
            'What changed since last time?',
        ]
    )
    usage = result.usage
    print(f'Cache writes: {usage.cache_write_tokens}')
    print(f'Cache reads: {usage.cache_read_tokens}')
```
Bedrock enforces a maximum of 4 cache points per request. Pydantic AI automatically manages this limit to ensure your requests always comply without errors.

Cache points can be placed in three locations:

1. **System Prompt** : Via`bedrock_cache_instructions` setting (adds cache point to last system prompt block)
2. **Tool Definitions** : Via`bedrock_cache_tool_definitions` setting (adds cache point to last tool definition)
3. **Messages** : Via`CachePoint` markers or`bedrock_cache_messages` setting (adds cache points to message content)

Each setting uses **at most 1 cache point**, but you can combine them.

When cache points from all sources (settings + `CachePoint` markers) exceed 4, Pydantic AI automatically removes excess cache points from **older message content** (keeping the most recent ones).

```
from pydantic_ai import Agent, CachePoint
from pydantic_ai.models.bedrock import BedrockModelSettings
agent = Agent(
    'bedrock:us.anthropic.claude-sonnet-4-5-20250929-v1:0',
    system_prompt='Instructions...',
    model_settings=BedrockModelSettings(
        bedrock_cache_instructions=True,      # 1 cache point
        bedrock_cache_tool_definitions=True,  # 1 cache point
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
```
**Key Points**:

- System and tool cache points are **always preserved**
- The cache point created by `bedrock_cache_messages` is**always preserved** (as it’s the newest message cache point)
- Additional `CachePoint` markers in messages are removed from oldest to newest when the limit is exceeded
- This ensures critical caching (instructions/tools) is maintained while still benefiting from message-level caching

You can provide a custom `BedrockProvider` via the `provider` argument. This is useful when you want to specify credentials directly or use a custom boto3 client:

```
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockConverseModel
from pydantic_ai.providers.bedrock import BedrockProvider
# Using AWS credentials directly
model = BedrockConverseModel(
    'anthropic.claude-sonnet-4-5-20250929-v1:0',
    provider=BedrockProvider(
        region_name='us-east-1',
        aws_access_key_id='your-access-key',
        aws_secret_access_key='your-secret-key',
    ),
)
agent = Agent(model)
...
```
You can also pass a pre-configured boto3 client:

```
import boto3
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockConverseModel
from pydantic_ai.providers.bedrock import BedrockProvider
# Using a pre-configured boto3 client
bedrock_client = boto3.client('bedrock-runtime', region_name='us-east-1')
model = BedrockConverseModel(
    'anthropic.claude-sonnet-4-5-20250929-v1:0',
    provider=BedrockProvider(bedrock_client=bedrock_client),
)
agent = Agent(model)
...
```
AWS Bedrock supports [custom application inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-create.html) for cost tracking and resource management. Set [`bedrock_inference_profile`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockModelSettings.bedrock_inference_profile) to route requests through an inference profile while keeping the base model name for detecting model capabilities:

```
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockConverseModel
from pydantic_ai.providers.bedrock import BedrockProvider
provider = BedrockProvider(region_name='us-east-2')
model = BedrockConverseModel(
    'us.anthropic.claude-opus-4-5-20251101-v1:0',
    provider=provider,
    settings={
        'bedrock_inference_profile': 'arn:aws:bedrock:us-east-2:123456789012:application-inference-profile/my-profile',
    },
)
agent = Agent(model)
```
Bedrock uses boto3’s built-in retry mechanisms. You can configure retry behavior by passing a custom boto3 client with retry settings:

```
import boto3
from botocore.config import Config
from pydantic_ai import Agent
from pydantic_ai.models.bedrock import BedrockConverseModel
from pydantic_ai.providers.bedrock import BedrockProvider
# Configure retry settings
config = Config(
    retries={
        'max_attempts': 5,
        'mode': 'adaptive'  # Recommended for rate limiting
    }
)
bedrock_client = boto3.client(
    'bedrock-runtime',
    region_name='us-east-1',
    config=config
)
model = BedrockConverseModel(
    'us.amazon.nova-micro-v1:0',
    provider=BedrockProvider(bedrock_client=bedrock_client),
)
agent = Agent(model)
```
- `'legacy'` (default): 5 attempts, basic retry behavior
- `'standard'` : 3 attempts, more comprehensive error coverage
- `'adaptive'` : 3 attempts with client-side rate limiting (recommended for handling`ThrottlingException` )

For more details on boto3 retry configuration, see the [AWS boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html).

[Amazon Bedrock Mantle](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html) serves OpenAI models (GPT-5.x and GPT-OSS) through an OpenAI-compatible API. Use the `bedrock-mantle:` prefix:

```
from pydantic_ai import Agent
agent = Agent('bedrock-mantle:openai.gpt-5.6-luna')
```
It requires the `bedrock-mantle` optional group:

The [`BedrockMantleProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.bedrock_mantle.BedrockMantleProvider) authenticates with the same AWS credentials as the [Converse route](#environment-variables) — a bearer token via `AWS_BEARER_TOKEN_BEDROCK`, or AWS access keys / profile via SigV4 — and derives its endpoint from `region_name` (or the `AWS_DEFAULT_REGION` / `AWS_REGION` environment variables).

The model name determines the endpoint family:

| Model name | Interface | 
|---|---|
| GPT-5.4+, e.g. `bedrock-mantle:openai.gpt-5.6-luna` | OpenAI Responses at `/openai/v1` | 
| GPT-OSS, e.g. `bedrock-mantle:openai.gpt-oss-120b` | OpenAI Responses at `/v1` | 
| GPT-OSS Safeguard, e.g. `bedrock-mantle:openai.gpt-oss-safeguard-20b` | OpenAI Chat Completions at `/v1` | 

To use a custom Mantle origin (for example a proxy), pass a `base_url` to [`BedrockMantleProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.bedrock_mantle.BedrockMantleProvider); its origin (with any `/openai/v1` or `/v1` suffix stripped) is used to route between the two endpoint families per model, just like `region_name`:

```
from pydantic_ai import Agent
from pydantic_ai.models.bedrock_mantle import BedrockMantleResponsesModel
from pydantic_ai.providers.bedrock_mantle import BedrockMantleProvider
provider = BedrockMantleProvider(base_url='https://bedrock-mantle.us-east-1.api.aws/openai/v1')
model = BedrockMantleResponsesModel('openai.gpt-5.6-luna', provider=provider)
agent = Agent(model)
```
Mantle models are served by Pydantic AI’s OpenAI model classes — [`BedrockMantleResponsesModel`](/docs/ai/api/models/bedrock_mantle/#pydantic_ai.models.bedrock_mantle.BedrockMantleResponsesModel) and [`BedrockMantleChatModel`](/docs/ai/api/models/bedrock_mantle/#pydantic_ai.models.bedrock_mantle.BedrockMantleChatModel) — so they accept the same settings as the direct [OpenAI](/docs/ai/models/openai) models ([`OpenAIResponsesModelSettings`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModelSettings) and [`OpenAIChatModelSettings`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModelSettings)).

The Converse-route features above — [prompt caching](#prompt-caching), [service tier](#service-tier), and [application inference profiles](#using-aws-application-inference-profiles) — are specific to the Converse API and don’t apply to the Mantle route.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/bedrock
