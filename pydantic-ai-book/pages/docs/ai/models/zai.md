---
type: Web Page
title: Z.AI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/zai
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Z.AI

To use `ZaiModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `zai` optional group:

To use [Z.AI](https://z.ai/) (Zhipu AI) through their API, go to [z.ai](https://z.ai/manage-apikey/apikey-list) and generate an API key.

For a list of available models, see the [Z.AI documentation](https://docs.z.ai/).

Once you have the API key, you can set it as an environment variable:

You can then use `ZaiModel` by name:

```
from pydantic_ai import Agent
agent = Agent('zai:glm-5')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.zai import ZaiModel
model = ZaiModel('glm-5')
agent = Agent(model)
...
```
Z.AI’s `glm-5.2`, `glm-5.1`, `glm-5`, `glm-4.7`, `glm-4.6` (hybrid thinking), and `glm-4.5` (interleaved thinking) models support thinking/reasoning mode, where the model produces reasoning content before the final response. This includes the `glm-4.6v` and `glm-4.5v` vision models. Configure this through the unified [`thinking`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.thinking) setting:

```
from pydantic_ai import Agent
from pydantic_ai.settings import ModelSettings
agent = Agent(
    'zai:glm-5',
    model_settings=ModelSettings(thinking=True),
)
...
```
`thinking=True` enables thinking and `thinking=False` disables it. On GLM-5.2, an explicit effort level (`'minimal'`/`'low'`/`'medium'`/`'high'`/`'xhigh'`) is forwarded to Z.AI as `reasoning_effort`; on other GLM models, which don’t expose effort granularity, the effort levels all collapse to enabled. Omit the field to use each model’s default behavior.

On thinking-capable models, reasoning content from prior assistant responses is **preserved by default** — no configuration required — for better multi-turn coherence and consistency with other providers. The complete, unmodified `reasoning_content` from prior turns is automatically sent back to the API by Pydantic AI.

If you instead want each turn to start fresh, **disable** it with `zai_clear_thinking=True` via the Z.AI-specific `ZaiModelSettings`:

```
from pydantic_ai import Agent
from pydantic_ai.models.zai import ZaiModelSettings
agent = Agent(
    'zai:glm-5',
    # Opt out of the default preserved thinking:
    model_settings=ZaiModelSettings(thinking=True, zai_clear_thinking=True),
)
...
```
See the [Z.AI thinking mode documentation](https://docs.z.ai/guides/capabilities/thinking-mode#preserved-thinking) for more details.

You can provide a custom [`Provider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.Provider) via the `provider` argument. In the simplest case, pass [`ZaiProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.zai.ZaiProvider) with just an API key. If you also want to customize the underlying `httpx.AsyncClient`, pass it when constructing the provider:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.zai import ZaiModel
from pydantic_ai.providers.zai import ZaiProvider
custom_http_client = AsyncClient(timeout=30)
model = ZaiModel(
    'glm-5',
    provider=ZaiProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```
If you do not need a custom HTTP client, omit the `http_client=custom_http_client` argument.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/zai
