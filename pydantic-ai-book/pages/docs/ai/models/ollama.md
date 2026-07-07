---
type: Web Page
title: Ollama | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/ollama
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Ollama

To use `OllamaModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openai` optional group:

Pydantic AI supports both self-hosted Ollama servers (running locally or remotely) and Ollama Cloud.

For servers running locally, use the `http://localhost:11434/v1` base URL. For Ollama Cloud, use `https://ollama.com/v1` and ensure an API key is set.

For backward compatibility, `OllamaModel` uses Ollama’s OpenAI-compatible Chat Completions API (`/v1/chat/completions`).

Set the `OLLAMA_BASE_URL` and (optionally) `OLLAMA_API_KEY` environment variables:

You can then use `OllamaModel` by name:

```
from pydantic_ai import Agent
agent = Agent('ollama:qwen3')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.ollama import OllamaModel
model = OllamaModel('qwen3')
agent = Agent(model)
...
```
You can provide a custom `Provider` via the `provider` argument:

```
from pydantic_ai import Agent
from pydantic_ai.models.ollama import OllamaModel
from pydantic_ai.providers.ollama import OllamaProvider
model = OllamaModel(
    'qwen3', provider=OllamaProvider(base_url='http://localhost:11434/v1')
)
agent = Agent(model)
...
```
For Ollama Cloud, use `base_url='https://ollama.com/v1'` and set the `OLLAMA_API_KEY` environment variable (or pass `api_key=` directly).

Self-hosted Ollama (v0.5.0+, released December 2024) enforces `response_format` with `json_schema` via `llama.cpp`’s grammar-constrained decoder, so `NativeOutput` produces schema-valid output at generation time:

```
from pydantic import BaseModel
from pydantic_ai import Agent
from pydantic_ai.models.ollama import OllamaModel
from pydantic_ai.output import NativeOutput
from pydantic_ai.providers.ollama import OllamaProvider
class CityLocation(BaseModel):
    city: str
    country: str
model = OllamaModel(
    'qwen3',
    provider=OllamaProvider(base_url='http://localhost:11434/v1'),
)
agent = Agent(model, output_type=NativeOutput(CityLocation))
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/ollama
