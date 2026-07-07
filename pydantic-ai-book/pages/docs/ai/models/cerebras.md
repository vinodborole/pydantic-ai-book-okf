---
type: Web Page
title: Cerebras | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/cerebras
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Cerebras

To use `CerebrasModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `cerebras` optional group:

To use Cerebras through their API, go to cloud.cerebras.ai and generate an API key.

For a list of available models, see the Cerebras models documentation.

Once you have the API key, you can set it as an environment variable:

You can then use `CerebrasModel` by name:

```
from pydantic_ai import Agent
agent = Agent('cerebras:llama-3.3-70b')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.cerebras import CerebrasModel
model = CerebrasModel('llama-3.3-70b')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.cerebras import CerebrasModel
from pydantic_ai.providers.cerebras import CerebrasProvider
model = CerebrasModel(
    'llama-3.3-70b', provider=CerebrasProvider(api_key='your-api-key')
)
agent = Agent(model)
...
```
You can also customize the `CerebrasProvider` with a custom `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.cerebras import CerebrasModel
from pydantic_ai.providers.cerebras import CerebrasProvider
custom_http_client = AsyncClient(timeout=30)
model = CerebrasModel(
    'llama-3.3-70b',
    provider=CerebrasProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/cerebras
