---
type: Web Page
title: OpenAI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/openai
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# OpenAI

To use OpenAI models or OpenAI-compatible APIs, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openai` optional group:

To use OpenAI models with the OpenAI API, go to platform.openai.com and follow your nose until you find the place to generate an API key.

Once you have the API key, you can set it as an environment variable:

The bare `'openai:'` prefix resolves to `OpenAIResponsesModel`, which uses the modern Responses API.

```
from pydantic_ai import Agent
agent = Agent('openai:gpt-5.2')
...
```
To pin to the legacy Chat Completions API instead, use the `'openai-chat:'` prefix, which resolves to `OpenAIChatModel`.

Or initialise the model directly with just the model name:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel
model = OpenAIResponsesModel('gpt-5.2')
agent = Agent(model)
...
```
By default, the model uses the `OpenAIProvider` with the `base_url` set to `https://api.openai.com/v1`.

If you want to pass parameters in code to the provider, you can programmatically instantiate the OpenAIProvider and pass it to the model:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIResponsesModel('gpt-5.2', provider=OpenAIProvider(api_key='your-api-key'))
agent = Agent(model)
...
```
`OpenAIProvider` also accepts a custom `AsyncOpenAI` client via the `openai_client` parameter, so you can customise the `organization`, `project`, `base_url` etc. as defined in the OpenAI API docs.

You could also use the `AsyncAzureOpenAI` client
to use the Azure OpenAI API. Note that the `AsyncAzureOpenAI` is a subclass of `AsyncOpenAI`.

```
from openai import AsyncAzureOpenAI
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
client = AsyncAzureOpenAI(
    azure_endpoint='...',
    api_version='2024-07-01-preview',
    api_key='your-api-key',
)
model = OpenAIChatModel(
    'gpt-5.2',
    provider=OpenAIProvider(openai_client=client),
)
agent = Agent(model)
...
```
You can customize model behavior using `OpenAIResponsesModelSettings`:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
model = OpenAIResponsesModel('gpt-5.2')
settings = OpenAIResponsesModelSettings(
    temperature=0.2,
    service_tier='flex',
)
agent = Agent(model, model_settings=settings)
...
```
OpenAI supports controlling the service tier to trade off latency and cost.
You can use the unified `service_tier` field or the provider-specific `openai_service_tier` field. Both accept `'auto'`, `'default'`, `'flex'`, and `'priority'`, passed through unchanged. `openai_service_tier` takes precedence over the unified field when both are set.

The features below are specific to the Responses API and only available on `OpenAIResponsesModel` (the default). For background on how the Responses API differs from Chat Completions, see the OpenAI API docs.

The Responses API has native tools that you can use instead of building your own:

- Web search: allow models to search the web for the latest information before generating a response.
- Code interpreter: allow models to write and run Python code in a sandboxed environment before generating a response.
- Image generation: allow models to generate images based on a text prompt.
- File search: allow models to search your files for relevant information before generating a response.
- Computer use: allow models to use a computer to perform tasks on your behalf.

Web search, Code interpreter, Image generation, and File search are natively supported through the Native tools feature.

Computer use can be enabled by passing an `openai.types.responses.ComputerToolParam` in the `openai_native_tools` setting on `OpenAIResponsesModelSettings`. It doesn’t currently generate `NativeToolCallPart` or `NativeToolReturnPart` parts in the message history, or streamed events; please submit an issue if you need native support for this native tool.

The Responses API supports referencing earlier model responses in a new request using a `previous_response_id` parameter, to ensure the full conversation state including reasoning items is kept in context without having to resend it. This is available through the `openai_previous_response_id` field in
`OpenAIResponsesModelSettings`.

When the field is set to `'auto'`, Pydantic AI automatically selects the most recent `provider_response_id` from the message history and omits messages that came before it, letting the OpenAI API reconstruct them from server-side state. The same chaining is applied inside a run across tool-call continuations and retries, so OpenAI never sees duplicate copies of the same messages.

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
model = OpenAIResponsesModel('gpt-5.2')
agent = Agent(model=model)
result1 = agent.run_sync('Tell me a joke.')
print(result1.output)
#> Did you hear about the toothpaste scandal? They called it Colgate.
model_settings = OpenAIResponsesModelSettings(openai_previous_response_id='auto')
result2 = agent.run_sync(
    'Explain?',
    message_history=result1.new_messages(),
    model_settings=model_settings
)
print(result2.output)
#> This is an excellent joke invented by Samuel Colvin, it needs no explanation.
```
As an alternative to passing `message_history`, you can pass a concrete `provider_response_id` from an earlier run as the seed. Pydantic AI uses the seed for the first request in the new run, then automatically chains to the response returned for that request on any subsequent in-run calls — so the chain still extends correctly if the run includes tool-call continuations or retries.

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
model = OpenAIResponsesModel('gpt-5.2')
agent = Agent(model=model)
result = agent.run_sync('The secret is 1234')
model_settings = OpenAIResponsesModelSettings(
    openai_previous_response_id=result.all_messages()[-1].provider_response_id
)
result = agent.run_sync('What is the secret code?', model_settings=model_settings)
print(result.output)
#> 1234
```
OpenAI’s Conversations API works with the Responses API to persist conversation state in a durable conversation object. If you already have an OpenAI conversation ID, pass it with `openai_conversation_id`:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
model = OpenAIResponsesModel('gpt-5.2')
agent = Agent(model=model)
model_settings = OpenAIResponsesModelSettings(openai_conversation_id='conv_...')
result = agent.run_sync('What did we discuss last time?', model_settings=model_settings)
print(result.output)
```
When a response belongs to a conversation, Pydantic AI stores the returned ID in `ModelResponse.provider_details['conversation_id']`. Setting `openai_conversation_id='auto'` uses the most recent same-provider conversation ID from the message history and sends only the new input items after that response.

When message-level `conversation_id` values are available, `auto` only reuses an OpenAI conversation from the current Pydantic AI conversation; pass a concrete OpenAI conversation ID to reuse one explicitly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIResponsesModel, OpenAIResponsesModelSettings
model = OpenAIResponsesModel('gpt-5.2')
agent = Agent(model=model)
model_settings = OpenAIResponsesModelSettings(openai_conversation_id='conv_...')
result = agent.run_sync('What did we discuss last time?', model_settings=model_settings)
follow_up_settings = OpenAIResponsesModelSettings(openai_conversation_id='auto')
result2 = agent.run_sync(
    'Summarize the next step.',
    message_history=result.new_messages(),
    model_settings=follow_up_settings,
)
print(result2.output)
```
Pydantic AI does not create OpenAI conversations for you. Use the OpenAI client to create the conversation, then pass its ID to `openai_conversation_id`. The `conversation` and `previous_response_id` parameters are mutually exclusive in the OpenAI API, so `openai_conversation_id` cannot be combined with `openai_previous_response_id`.

The Responses API supports compacting message history to reduce token usage in long conversations. Compaction produces an encrypted summary that replaces older messages while preserving context.

The easiest way to enable compaction is with the `OpenAICompaction` capability:

By default, `OpenAICompaction` runs in **stateful mode**: it configures OpenAI’s server-side auto-compaction via the `context_management` field on the regular `/responses` request, and OpenAI triggers compaction whenever the input token count crosses a threshold it manages for you. This mode is compatible with `openai_previous_response_id='auto'` and `openai_conversation_id`.

To override the threshold, pass `token_threshold`:

As an alternative, `OpenAICompaction` supports a **stateless mode** (`stateless=True`) that calls the stateless `/responses/compact` endpoint via a `before_model_request` hook. Use this in ZDR environments where OpenAI must not retain conversation data, when using `openai_store=False`, or when you need explicit out-of-band control over when compaction runs. Stateless mode requires you to specify either a `message_count_threshold` or a custom `trigger` callable:

The mode is inferred from which parameters you pass: supplying `message_count_threshold` or `trigger` implies stateless mode, otherwise stateful mode is used. You can also pass `stateless=True` or `stateless=False` explicitly. Mixing parameters from different modes raises `UserError`.

For lower-level use cases, you can call `compact_messages` directly on the model.

If you need the Chat Completions API instead of the default Responses API, pin to it with the `'openai-chat:'` prefix or `OpenAIChatModel`:

```
from pydantic_ai import Agent
agent = Agent('openai-chat:gpt-5.2')
...
```
```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
model = OpenAIChatModel('gpt-5.2')
agent = Agent(model)
...
```
`OpenAIChatModel` is also what backs every OpenAI-compatible provider below — they all speak the Chat Completions wire format, so the same model class applies.

Many providers and models are compatible with the OpenAI API, and can be used with `OpenAIChatModel` in Pydantic AI.
Before getting started, check the installation and configuration instructions above.

To use another OpenAI-compatible API, you can set the `OPENAI_BASE_URL` and `OPENAI_API_KEY` environment variables, or make use of the `base_url` and `api_key` arguments from `OpenAIProvider`:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIChatModel(
    'model_name',
    provider=OpenAIProvider(
        base_url='https://<openai-compatible-api-endpoint>', api_key='your-api-key'
    ),
)
agent = Agent(model)
...
```
Various providers also have their own provider classes so that you don’t need to specify the base URL yourself and you can use the standard `<PROVIDER>_API_KEY` environment variable to set the API key.
When a provider has its own provider class, you can use the `Agent("<provider>:<model>")` shorthand, e.g. `Agent("deepseek:deepseek-chat")` or `Agent("moonshotai:kimi-k2-0711-preview")`, instead of building the `OpenAIChatModel` explicitly. Similarly, you can pass the provider name as a string to the `provider` argument on `OpenAIChatModel` instead of instantiating the provider class explicitly.

Sometimes, the provider or model you’re using will have slightly different requirements than OpenAI’s API or models, like having different restrictions on JSON schemas for tool definitions, or not supporting tool definitions to be marked as strict.

When using an alternative provider class provided by Pydantic AI, an appropriate model profile is typically selected automatically based on the model name.
If the model you’re using is not working correctly out of the box, you can tweak various aspects of how model requests are constructed by providing your own `ModelProfile` (for behaviors shared among all model classes) or `OpenAIModelProfile` (for behaviors specific to `OpenAIChatModel`):

```
from pydantic_ai import Agent, InlineDefsJsonSchemaTransformer
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.profiles.openai import OpenAIModelProfile
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIChatModel(
    'model_name',
    provider=OpenAIProvider(
        base_url='https://<openai-compatible-api-endpoint>.com', api_key='your-api-key'
    ),
    profile=OpenAIModelProfile(
        json_schema_transformer=InlineDefsJsonSchemaTransformer,  # Supported by any model class via the base ModelProfile
        openai_supports_strict_tool_definition=False,  # Supported by OpenAIChatModel and OpenAIResponsesModel
        openai_chat_supports_multiple_system_messages=False,  # Supported by OpenAIChatModel only — for strict providers (e.g. some vLLM/LiteLLM setups) that require exactly one initial system message
        openai_chat_supports_max_completion_tokens=False,  # Supported by OpenAIChatModel only — for providers (e.g. OpenRouter) that only accept the older `max_tokens` field instead of `max_completion_tokens`
    )
)
agent = Agent(model)
```
Some models are served with a chat template (applied server-side, for example by vLLM, LiteLLM, or TGI) that accepts only a single system message at the start of the conversation and rejects additional ones. Sending more than one fails with a `400` error such as `System message must be at the beginning.` or `Conversation roles must alternate ...`, seen with some newer Qwen, Mistral, Gemma, and Command-R models. It’s easy to hit without intending to, since more than one leading system message can be produced in several ways.

Set `openai_chat_supports_multiple_system_messages=False` on the model’s `OpenAIModelProfile` (as shown above) to merge the leading run of system messages into one, joined with two newlines, before the request is sent. The merge is lossless, so it’s safe to enable whenever a backend rejects multiple system messages.

To use the DeepSeek provider, first create an API key by following the Quick Start guide.

You can then set the `DEEPSEEK_API_KEY` environment variable and use `DeepSeekProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('deepseek:deepseek-chat')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.deepseek import DeepSeekProvider
model = OpenAIChatModel(
    'deepseek-chat',
    provider=DeepSeekProvider(api_key='your-deepseek-api-key'),
)
agent = Agent(model)
...
```
You can also customize any provider with a custom `http_client`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.deepseek import DeepSeekProvider
custom_http_client = AsyncClient(timeout=30)
model = OpenAIChatModel(
    'deepseek-chat',
    provider=DeepSeekProvider(
        api_key='your-deepseek-api-key', http_client=custom_http_client
    ),
)
agent = Agent(model)
...
```
To use Qwen models via Alibaba Cloud Model Studio (DashScope), you can set the `ALIBABA_API_KEY` (or `DASHSCOPE_API_KEY`) environment variable and use `AlibabaProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('alibaba:qwen-max')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.alibaba import AlibabaProvider
model = OpenAIChatModel(
    'qwen-max',
    provider=AlibabaProvider(api_key='your-api-key'),
)
agent = Agent(model)
...
```
The `AlibabaProvider` uses the international DashScope compatible endpoint `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` by default. You can override this by passing a custom `base_url`:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.alibaba import AlibabaProvider
model = OpenAIChatModel(
    'qwen-max',
    provider=AlibabaProvider(
        api_key='your-api-key',
        base_url='https://dashscope.aliyuncs.com/compatible-mode/v1',  # China region
    ),
)
agent = Agent(model)
...
```
See Ollama for dedicated Ollama documentation, including structured output and Ollama Cloud limitations.

To use Azure AI Foundry as your provider, set `AZURE_OPENAI_ENDPOINT` to a URL whose path ends in `/v1` (for example `https://<resource>.openai.azure.com/openai/v1/` or `https://<resource>.services.ai.azure.com/openai/v1/`), set `AZURE_OPENAI_API_KEY`, and use `AzureProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('azure:gpt-5.2')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.azure import AzureProvider
model = OpenAIChatModel(
    'gpt-5.2',
    provider=AzureProvider(
        azure_endpoint='https://your-resource.openai.azure.com/openai/v1/',
        api_key='your-api-key',
    ),
)
agent = Agent(model)
...
```
This targets the Azure OpenAI v1 API, which Microsoft recommends for all new projects. It also pairs naturally with the Responses API — see Using Azure with the Responses API below.

`AzureProvider` also recognises Azure AI Foundry serverless model deployments at `https://<model>.<region>.models.ai.azure.com` and connects to them the same way.

If your resource still uses the dated `api-version` API, pass `api_version` (or set the `OPENAI_API_VERSION` environment variable) and point `azure_endpoint` at the resource root instead:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.azure import AzureProvider
model = OpenAIChatModel(
    'gpt-5.2',
    provider=AzureProvider(
        azure_endpoint='https://your-resource.openai.azure.com/',
        api_version='2024-12-01-preview',
        api_key='your-api-key',
    ),
)
agent = Agent(model)
...
```
Azure AI Foundry also supports the OpenAI Responses API through `OpenAIResponsesModel`. This is particularly recommended when working with document inputs (`DocumentUrl` and `BinaryContent`), as Azure’s Chat Completions API does not support these input types.

## Document processing with Azure using Responses API

```
from pydantic_ai import Agent, BinaryContent
from pydantic_ai.models.openai import OpenAIResponsesModel
from pydantic_ai.providers.azure import AzureProvider
pdf_bytes = b'%PDF-1.4 ...'  # Your PDF content
model = OpenAIResponsesModel(
    'gpt-5.2',
    provider=AzureProvider(
        azure_endpoint='https://your-resource.openai.azure.com/openai/v1/',
        api_key='your-api-key',
    ),
)
agent = Agent(model)
result = agent.run_sync([
    'Summarize this document',
    BinaryContent(data=pdf_bytes, media_type='application/pdf'),
])
```
To use Vercel’s AI Gateway, first follow the documentation instructions on obtaining an API key or OIDC token.

You can set the `VERCEL_AI_GATEWAY_API_KEY` and `VERCEL_OIDC_TOKEN` environment variables and use `VercelProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('vercel:anthropic/claude-sonnet-4-5')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.vercel import VercelProvider
model = OpenAIChatModel(
    'anthropic/claude-sonnet-4-5',
    provider=VercelProvider(api_key='your-vercel-ai-gateway-api-key'),
)
agent = Agent(model)
...
```
Create an API key in the Moonshot Console.

You can set the `MOONSHOTAI_API_KEY` environment variable and use `MoonshotAIProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('moonshotai:kimi-k2-0711-preview')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.moonshotai import MoonshotAIProvider
model = OpenAIChatModel(
    'kimi-k2-0711-preview',
    provider=MoonshotAIProvider(api_key='your-moonshot-api-key'),
)
agent = Agent(model)
...
```
To use GitHub Models, you’ll need a GitHub personal access token with the `models: read` permission.

You can set the `GITHUB_API_KEY` environment variable and use `GitHubProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('github:xai/grok-3-mini')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.github import GitHubProvider
model = OpenAIChatModel(
    'xai/grok-3-mini',  # GitHub Models uses prefixed model names
    provider=GitHubProvider(api_key='your-github-token'),
)
agent = Agent(model)
...
```
GitHub Models supports various model families with different prefixes. You can see the full list on the GitHub Marketplace or the public catalog endpoint.

Follow the Perplexity getting started guide to create an API key, then initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIChatModel(
    'sonar-pro',
    provider=OpenAIProvider(
        base_url='https://api.perplexity.ai',
        api_key='your-perplexity-api-key',
    ),
)
agent = Agent(model)
...
```
Go to Fireworks.AI and create an API key in your account settings.

You can set the `FIREWORKS_API_KEY` environment variable and use `FireworksProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('fireworks:accounts/fireworks/models/qwq-32b')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.fireworks import FireworksProvider
model = OpenAIChatModel(
    'accounts/fireworks/models/qwq-32b',  # model library available at https://fireworks.ai/models
    provider=FireworksProvider(api_key='your-fireworks-api-key'),
)
agent = Agent(model)
...
```
Go to Together.ai and create an API key in your account settings.

You can set the `TOGETHER_API_KEY` environment variable and use `TogetherProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('together:meta-llama/Llama-3.3-70B-Instruct-Turbo-Free')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.together import TogetherProvider
model = OpenAIChatModel(
    'meta-llama/Llama-3.3-70B-Instruct-Turbo-Free',  # model library available at https://www.together.ai/models
    provider=TogetherProvider(api_key='your-together-api-key'),
)
agent = Agent(model)
...
```
To use Heroku AI, first create an API key.

You can set the `HEROKU_INFERENCE_KEY` and (optionally) `HEROKU_INFERENCE_URL` environment variables and use `HerokuProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('heroku:claude-sonnet-4-5')
...
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.heroku import HerokuProvider
model = OpenAIChatModel(
    'claude-sonnet-4-5',
    provider=HerokuProvider(api_key='your-heroku-inference-key'),
)
agent = Agent(model)
...
```
To use LiteLLM, set the configs as outlined in the doc. In `LiteLLMProvider`, you can pass `api_base` and `api_key`. The value of these configs will depend on your setup. For example, if you are using OpenAI models, then you need to pass `https://api.openai.com/v1` as the `api_base` and your OpenAI API key as the `api_key`. If you are using a LiteLLM proxy server running on your local machine, then you need to pass `http://localhost:<port>` as the `api_base` and your LiteLLM API key (or a placeholder) as the `api_key`.

To use custom LLMs, use `custom/` prefix in the model name.

Once you have the configs, use the `LiteLLMProvider` as follows:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.litellm import LiteLLMProvider
model = OpenAIChatModel(
    'openai/gpt-5.2',
    provider=LiteLLMProvider(
        api_base='<api-base-url>',
        api_key='<api-key>'
    )
)
agent = Agent(model)
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
...
```
Go to Nebius AI Studio and create an API key.

You can set the `NEBIUS_API_KEY` environment variable and use `NebiusProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('nebius:Qwen/Qwen3-32B-fast')
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.nebius import NebiusProvider
model = OpenAIChatModel(
    'Qwen/Qwen3-32B-fast',
    provider=NebiusProvider(api_key='your-nebius-api-key'),
)
agent = Agent(model)
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
To use OVHcloud AI Endpoints, you need to create a new API key. To do so, go to the OVHcloud manager, then in Public Cloud > AI Endpoints > API keys. Click on `Create a new API key` and copy your new key.

You can explore the catalog to find which models are available.

You can set the `OVHCLOUD_API_KEY` environment variable and use `OVHcloudProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('ovhcloud:gpt-oss-120b')
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
If you need to configure the provider, you can use the `OVHcloudProvider` class:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.ovhcloud import OVHcloudProvider
model = OpenAIChatModel(
    'gpt-oss-120b',
    provider=OVHcloudProvider(api_key='your-api-key'),
)
agent = Agent(model)
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
To use SambaNova Cloud, you need to obtain an API key from the SambaNova Cloud dashboard.

SambaNova provides access to multiple model families including Meta Llama, DeepSeek, Qwen, and Mistral models with fast inference speeds.

You can set the `SAMBANOVA_API_KEY` environment variable and use `SambaNovaProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('sambanova:Meta-Llama-3.1-8B-Instruct')
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
Or initialise the model and provider directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.sambanova import SambaNovaProvider
model = OpenAIChatModel(
    'Meta-Llama-3.1-8B-Instruct',
    provider=SambaNovaProvider(api_key='your-api-key'),
)
agent = Agent(model)
result = agent.run_sync('What is the capital of France?')
print(result.output)
#> The capital of France is Paris.
```
For a complete list of available models, see the SambaNova supported models documentation.

You can customize the base URL if needed:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.sambanova import SambaNovaProvider
model = OpenAIChatModel(
    'DeepSeek-R1-0528',
    provider=SambaNovaProvider(
        api_key='your-api-key',
        base_url='https://custom.endpoint.com/v1',
    ),
)
agent = Agent(model)
...
```
Atlas Cloud is an OpenAI-compatible API gateway that provides access to 300+ models from a single endpoint, including DeepSeek, Qwen, Claude, GPT, and Gemini.

Atlas Cloud doesn’t have a dedicated provider class, so you can use it with `OpenAIProvider` by setting the `base_url` and `api_key`:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIChatModel(
    'deepseek-ai/deepseek-v4-pro',
    provider=OpenAIProvider(
        base_url='https://api.atlascloud.ai/v1',
        api_key='your-atlas-cloud-api-key',
    ),
)
agent = Agent(model)
...
```
Rapid-MLX is an OpenAI-compatible inference server for Apple Silicon, built on Apple’s MLX framework.

The server listens on `http://localhost:8000/v1` and implements the OpenAI chat completions API, so you can point `OpenAIProvider` at it:

```
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
rapid_mlx_model = OpenAIChatModel(
    model_name='default',
    provider=OpenAIProvider(
        base_url='http://localhost:8000/v1',
        api_key='not-needed',
    ),
)
agent = Agent(rapid_mlx_model)
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/openai
