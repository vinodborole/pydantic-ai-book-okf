---
type: Web Page
title: Mistral | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/mistral
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Mistral

To use `MistralModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `mistral` optional group:

To use [Mistral](https://mistral.ai) through their API, go to [console.mistral.ai/api-keys/](https://console.mistral.ai/api-keys/) and follow your nose until you find the place to generate an API key.

`LatestMistralModelNames` contains a list of the most popular Mistral models.

Once you have the API key, you can set it as an environment variable:

You can then use `MistralModel` by name:

```
from pydantic_ai import Agent
agent = Agent('mistral:mistral-large-latest')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.mistral import MistralModel
model = MistralModel('mistral-small-latest')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.mistral import MistralModel
from pydantic_ai.providers.mistral import MistralProvider
model = MistralModel(
    'mistral-large-latest', provider=MistralProvider(api_key='your-api-key', base_url='https://<mistral-provider-endpoint>')
)
agent = Agent(model)
...
```
You can also customize the provider with a custom `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.mistral import MistralModel
from pydantic_ai.providers.mistral import MistralProvider
custom_http_client = AsyncClient(timeout=30)
model = MistralModel(
    'mistral-large-latest',
    provider=MistralProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/mistral
