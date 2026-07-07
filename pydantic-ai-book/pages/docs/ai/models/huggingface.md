---
type: Web Page
title: Hugging Face | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/huggingface
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Hugging Face

Hugging Face is an AI platform with all major open source models, datasets, MCPs, and demos. You can use Inference Providers to run open source models like DeepSeek R1 on scalable serverless infrastructure.

To use `HuggingFaceModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `huggingface` optional group:

To use Hugging Face inference, you’ll need to set up an account which will give you free tier allowance on Inference Providers. To setup inference, follow these steps:

- Go to Hugging Face and sign up for an account.
- Create a new access token in Hugging Face.
- Set the `HF_TOKEN`environment variable to the token you just created.

Once you have a Hugging Face access token, you can set it as an environment variable:

You can then use `HuggingFaceModel` by name:

```
from pydantic_ai import Agent
agent = Agent('huggingface:Qwen/Qwen3-235B-A22B')
...
```
Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.huggingface import HuggingFaceModel
model = HuggingFaceModel('Qwen/Qwen3-235B-A22B')
agent = Agent(model)
...
```
By default, the `HuggingFaceModel` uses the
`HuggingFaceProvider` that will select automatically
the first of the inference providers (Cerebras, Together AI, Cohere..etc) available for the model, sorted by your
preferred order in https://hf.co/settings/inference-providers.

If you want to pass parameters in code to the provider, you can programmatically instantiate the
`HuggingFaceProvider` and pass it to the model:

```
from pydantic_ai import Agent
from pydantic_ai.models.huggingface import HuggingFaceModel
from pydantic_ai.providers.huggingface import HuggingFaceProvider
model = HuggingFaceModel('Qwen/Qwen3-235B-A22B', provider=HuggingFaceProvider(api_key='hf_token', provider_name='nebius'))
agent = Agent(model)
...
```
`HuggingFaceProvider` also accepts a custom
`AsyncInferenceClient` client via the `hf_client` parameter, so you can customise
the `headers`, `bill_to` (billing to an HF organization you’re a member of), `base_url` etc. as defined in the
Hugging Face Hub python library docs.

```
from huggingface_hub import AsyncInferenceClient
from pydantic_ai import Agent
from pydantic_ai.models.huggingface import HuggingFaceModel
from pydantic_ai.providers.huggingface import HuggingFaceProvider
client = AsyncInferenceClient(
    bill_to='openai',
    api_key='hf_token',
    provider='fireworks-ai',
)
model = HuggingFaceModel(
    'Qwen/Qwen3-235B-A22B',
    provider=HuggingFaceProvider(hf_client=client),
)
agent = Agent(model)
...
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/huggingface
