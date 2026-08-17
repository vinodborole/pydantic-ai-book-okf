---
type: Web Page
title: Snowflake Cortex | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/snowflake
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Snowflake Cortex

Snowflake Cortex rejects [`service_tier`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.service_tier) with an error rather than ignoring it, so leave that setting unset on Snowflake models.
To use [`SnowflakeModel`](/docs/ai/api/models/snowflake/#pydantic_ai.models.snowflake.SnowflakeModel), you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `snowflake` optional group:

[Snowflake Cortex](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-rest-api) serves Claude, GPT, Llama, Mistral, DeepSeek, and Snowflake’s own models through a REST API hosted in your Snowflake account, so data never leaves the Snowflake security perimeter.

To use it, you need your [Snowflake account identifier](https://docs.snowflake.com/en/user-guide/admin-account-identifier) (e.g. `myorg-myaccount`) and a token: a [programmatic access token](https://docs.snowflake.com/en/user-guide/programmatic-access-tokens) (PAT), OAuth token, or key-pair JWT. The role the request runs as — the role a PAT is restricted to, or otherwise your user’s default role — must have the `SNOWFLAKE.CORTEX_USER` database role, which is [granted to `PUBLIC` by default](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql#required-privileges).

For a list of available models, see the [Cortex REST API documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-rest-api). [Fine-tuned models](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-finetuning) can be referenced as `database.schema.model`.

Once you have the account identifier and token, you can set them as environment variables:

You can then use [`SnowflakeModel`](/docs/ai/api/models/snowflake/#pydantic_ai.models.snowflake.SnowflakeModel) by name:

```
from pydantic_ai import Agent
agent = Agent('snowflake:claude-sonnet-4-6')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.snowflake import SnowflakeModel
model = SnowflakeModel('claude-sonnet-4-6')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.snowflake import SnowflakeModel
from pydantic_ai.providers.snowflake import SnowflakeProvider
model = SnowflakeModel(
    'claude-sonnet-4-6',
    provider=SnowflakeProvider(account='myorg-myaccount', token='your-token'),
)
agent = Agent(model)
...
```
You can also customize the [`SnowflakeProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.snowflake.SnowflakeProvider) with a custom `base_url` (e.g. when connecting through [private connectivity](https://docs.snowflake.com/en/user-guide/private-snowflake-service)) or `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.snowflake import SnowflakeModel
from pydantic_ai.providers.snowflake import SnowflakeProvider
model = SnowflakeModel(
    'claude-sonnet-4-6',
    provider=SnowflakeProvider(
        base_url='https://myorg-myaccount.privatelink.snowflakecomputing.com/api/v2/cortex/v1',
        token='your-token',
        http_client=AsyncClient(timeout=30),
    ),
)
agent = Agent(model)
...
```
Cortex only supports tool calling and structured output for OpenAI (`openai-*`) and Claude (`claude-*`) models; for other model families, structured output falls back to [prompted output](/docs/ai/core-concepts/output/#prompted-output).

To enable thinking on Claude models, use the unified `thinking`[model setting](/docs/ai/core-concepts/agent/#model-run-settings), or set [`SnowflakeModelSettings.snowflake_reasoning`](/docs/ai/api/models/snowflake/#pydantic_ai.models.snowflake.SnowflakeModelSettings.snowflake_reasoning) directly to control the reasoning token budget:

```
from pydantic_ai import Agent
from pydantic_ai.models.snowflake import SnowflakeModel, SnowflakeModelSettings
agent = Agent(
    SnowflakeModel('claude-sonnet-4-6'),
    model_settings=SnowflakeModelSettings(snowflake_reasoning={'max_tokens': 4096}),
)
...
```
On OpenAI models, use the unified `thinking` setting or [`openai_reasoning_effort`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModelSettings.openai_reasoning_effort).

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/snowflake
