---
type: Web Page
title: Groq | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/groq
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Groq

To use `GroqModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `groq` optional group:

To use [Groq](https://groq.com/) through their API, go to [console.groq.com/keys](https://console.groq.com/keys) and follow your nose until you find the place to generate an API key.

`GroqModelName` contains a list of available Groq models.

Once you have the API key, you can set it as an environment variable:

You can then use `GroqModel` by name:

```
from pydantic_ai import Agent
agent = Agent('groq:llama-3.3-70b-versatile')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.groq import GroqModel
model = GroqModel('llama-3.3-70b-versatile')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.groq import GroqModel
from pydantic_ai.providers.groq import GroqProvider
model = GroqModel(
    'llama-3.3-70b-versatile', provider=GroqProvider(api_key='your-api-key')
)
agent = Agent(model)
...
```
You can also customize the `GroqProvider` with a custom `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.groq import GroqModel
from pydantic_ai.providers.groq import GroqProvider
custom_http_client = AsyncClient(timeout=30)
model = GroqModel(
    'llama-3.3-70b-versatile',
    provider=GroqProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/groq
