---
type: Web Page
title: Cohere | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/cohere
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Cohere

To use `CohereModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `cohere` optional group:

To use [Cohere](https://cohere.com/) through their API, go to [dashboard.cohere.com/api-keys](https://dashboard.cohere.com/api-keys) and follow your nose until you find the place to generate an API key.

`CohereModelName` contains a list of the most popular Cohere models.

Once you have the API key, you can set it as an environment variable:

You can then use `CohereModel` by name:

```
from pydantic_ai import Agent
agent = Agent('cohere:command-r7b-12-2024')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.cohere import CohereModel
model = CohereModel('command-r7b-12-2024')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.cohere import CohereModel
from pydantic_ai.providers.cohere import CohereProvider
model = CohereModel('command-r7b-12-2024', provider=CohereProvider(api_key='your-api-key'))
agent = Agent(model)
...
```
You can also customize the `CohereProvider` with a custom `http_client`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.cohere import CohereModel
from pydantic_ai.providers.cohere import CohereProvider
custom_http_client = AsyncClient(timeout=30)
model = CohereModel(
    'command-r7b-12-2024',
    provider=CohereProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```
You can customize model behavior using [`CohereModelSettings`](/docs/ai/api/models/cohere/#pydantic_ai.models.cohere.CohereModelSettings):

```
from pydantic_ai import Agent
from pydantic_ai.models.cohere import CohereModel, CohereModelSettings
model = CohereModel('command-r7b-12-2024')
settings = CohereModelSettings(
    temperature=0.2,
    top_k=40,
)
agent = Agent(model, model_settings=settings)
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/cohere
