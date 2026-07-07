---
type: Web Page
title: Google | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/google
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

The `GoogleModel` is a model that uses the `google-genai` package under the hood to
access Google’s Gemini models via both the Gemini API and Google Cloud (formerly known as Vertex AI).

Two providers wrap those endpoints:

- `GoogleProvider`— the Gemini API (Google AI Studio), surfaced under the- `'google:'`prefix.
- `GoogleCloudProvider`— Google Cloud (formerly known as Vertex AI), surfaced under the- `'google-cloud:'`prefix.

To use `GoogleModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `google` optional group:

`GoogleModel` lets you use Google’s Gemini models through their Gemini API (`generativelanguage.googleapis.com`) or Google Cloud (`*-aiplatform.googleapis.com`, formerly known as Vertex AI).

To use Gemini via the Gemini API, go to aistudio.google.com and create an API key.

Once you have the API key, set it as an environment variable:

You can then use `GoogleModel` by name:

```
from pydantic_ai import Agent
agent = Agent('google:gemini-3-pro-preview')
...
```
Or you can explicitly create the provider:

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google import GoogleProvider
provider = GoogleProvider(api_key='your-api-key')
model = GoogleModel('gemini-3-pro-preview', provider=provider)
agent = Agent(model)
...
```
If you are an enterprise user, you can also use `GoogleModel` to access Gemini via Google Cloud (formerly known as Vertex AI).

This interface has a number of advantages over the Gemini API:

- The Google Cloud API comes with more enterprise readiness guarantees.
- You can purchase provisioned throughput with Google Cloud to guarantee capacity.
- If you’re running Pydantic AI inside Google Cloud, you don’t need to set up authentication, it should “just work”.
- You can decide which region to use, which might be important from a regulatory perspective, and might improve latency.

You can authenticate using application default credentials, a service account, or an API key.

Whichever way you authenticate, you’ll need to have the Vertex AI API (now branded as Google Cloud AI) enabled in your Google Cloud account.

If you have the `gcloud` CLI installed and configured, you can use the `GoogleCloudProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('google-cloud:gemini-3-pro-preview')
...
```
Or you can explicitly create the provider and model:

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google_cloud import GoogleCloudProvider
provider = GoogleCloudProvider()
model = GoogleModel('gemini-3-pro-preview', provider=provider)
agent = Agent(model)
...
```
To use a service account JSON file, explicitly create the provider and model:

To use Google Cloud with an API key, create a key and set it as an environment variable:

You can then use `GoogleModel` via the `GoogleCloudProvider` by name:

```
from pydantic_ai import Agent
agent = Agent('google-cloud:gemini-3-pro-preview')
...
```
Or you can explicitly create the provider and model:

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google_cloud import GoogleCloudProvider
provider = GoogleCloudProvider(api_key='your-api-key')
model = GoogleModel('gemini-3-pro-preview', provider=provider)
agent = Agent(model)
...
```
You can specify the location and/or project when using Google Cloud:

The unified `service_tier` field works on both Google subsystems, with `google_cloud_service_tier` available for finer Google Cloud routing control. The provider-specific field wins when both are set.

**Gemini API** — sent as the request’s `service_tier` field:

| `service_tier` | Sent to Gemini API | 
|---|---|
| `'auto'` | (omitted — server default) | 
| `'default'` | `'standard'` | 
| `'flex'` | `'flex'` | 
| `'priority'` | `'priority'` | 

**Google Cloud** — sent as HTTP routing headers; `'flex'` and `'priority'` always pick the **PT-with-spillover** variant, so customers with Provisioned Throughput (PT) keep using their reserved capacity first:

| `service_tier` | Google Cloud routing headers | Effective behavior | 
|---|---|---|
| `'auto'`/`'default'` | (none) | PT first, then standard on-demand spillover | 
| `'flex'` | `X-Vertex-AI-LLM-Shared-Request-Type: flex` | PT first, then Flex PayGo spillover | 
| `'priority'` | `X-Vertex-AI-LLM-Shared-Request-Type: priority` | PT first, then Priority PayGo spillover | 

To bypass PT entirely (or use it exclusively, or any of the other Google Cloud-specific routing combinations) set `google_cloud_service_tier` directly — the unified field is intentionally limited to the safe PT-with-spillover variants.

**Google Cloud — full set of routing values**

The full `google_cloud_service_tier` values map to these HTTP headers:

- `'pt_only'`: PT only (- `X-Vertex-AI-LLM-Request-Type: dedicated`).
- `'pt_then_flex'`: PT when quota allows, then Flex PayGo spillover (- `X-Vertex-AI-LLM-Shared-Request-Type: flex`).
- `'pt_then_priority'`: PT when quota allows, then Priority PayGo spillover (- `X-Vertex-AI-LLM-Shared-Request-Type: priority`).
- `'on_demand'`: Standard on-demand only (- `X-Vertex-AI-LLM-Request-Type: shared`).
- `'flex_only'`: Flex PayGo only (- `X-Vertex-AI-LLM-Request-Type: shared`and- `X-Vertex-AI-LLM-Shared-Request-Type: flex`).
- `'priority_only'`: Priority PayGo only (- `X-Vertex-AI-LLM-Request-Type: shared`and- `X-Vertex-AI-LLM-Shared-Request-Type: priority`).

**Example**

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
from pydantic_ai.providers.google_cloud import GoogleCloudProvider
provider = GoogleCloudProvider(location='global')
model = GoogleModel('gemini-3-flash-preview', provider=provider)
agent = Agent(model)
result = agent.run_sync(
    'Hello!',
    model_settings=GoogleModelSettings(google_cloud_service_tier='pt_then_flex'),
)
```
Swap `'pt_then_flex'` for any `GoogleCloudServiceTier` value — e.g. `'pt_then_priority'` for Priority PayGo spillover, or `'flex_only'` / `'priority_only'` to bypass PT entirely.

After the request, inspect `ModelResponse` `provider_details.get('traffic_type')` (e.g. `ON_DEMAND_FLEX`, `ON_DEMAND_PRIORITY`) to see which tier served it, when the API returns it.

You can access models from the Model Garden that support the `generateContent` API and are available under your Google Cloud project, including but not limited to Gemini, using one of the following `model_name` patterns:

- `{model_id}`for Gemini models
- `{publisher}/{model_id}`
- `publishers/{publisher}/models/{model_id}`
- `projects/{project}/locations/{location}/publishers/{publisher}/models/{model_id}`

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google_cloud import GoogleCloudProvider
provider = GoogleCloudProvider(
    project='your-google-cloud-project-id',
    location='us-central1',  # the region where the model is available
)
model = GoogleModel('meta/llama-3.3-70b-instruct-maas', provider=provider)
agent = Agent(model)
...
```
You can customize the `GoogleProvider` with a custom `httpx.AsyncClient`:

```
from httpx import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google import GoogleProvider
custom_http_client = AsyncClient(timeout=30)
model = GoogleModel(
    'gemini-3-pro-preview',
    provider=GoogleProvider(api_key='your-api-key', http_client=custom_http_client),
)
agent = Agent(model)
...
```
By default, the `google-genai` SDK does not retry requests that fail with a transient HTTP error. You can enable retries by passing a `HttpRetryOptions` instance to the `retry_options` argument of `GoogleProvider` or `GoogleCloudProvider`:

```
from google.genai.types import HttpRetryOptions
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel
from pydantic_ai.providers.google import GoogleProvider
retry_options = HttpRetryOptions(
    attempts=4,
    initial_delay=1.0,
    max_delay=60.0,
    http_status_codes=[408, 429, 500, 502, 503, 504],
)
model = GoogleModel(
    'gemini-3-pro-preview',
    provider=GoogleProvider(api_key='your-api-key', retry_options=retry_options),
)
agent = Agent(model)
...
```
This passes the options through to the SDK’s `HttpOptions.retry_options`. See the Vertex AI retry strategy documentation for guidance on choosing values.

`GoogleModel` supports multi-modal input, including documents, images, audio, and video.

YouTube video URLs can be passed directly to Google models:

Files can be uploaded via the Files API and passed as URLs:

See the input documentation for more details and examples.

You can customize model behavior using `GoogleModelSettings`:

```
from google.genai.types import HarmBlockThreshold, HarmCategory
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
settings = GoogleModelSettings(
    temperature=0.2,
    max_tokens=1024,
    top_k=40,
    google_safety_settings=[
        {
            'category': HarmCategory.HARM_CATEGORY_HATE_SPEECH,
            'threshold': HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
        }
    ]
)
model = GoogleModel('gemini-3-pro-preview')
agent = Agent(model, model_settings=settings)
...
```
Use the provider-agnostic `Thinking` capability to enable thinking:

```
from pydantic_ai import Agent
from pydantic_ai.capabilities import Thinking
agent = Agent('google:gemini-3.5-flash', capabilities=[Thinking(effort='medium')])
...
```
For advanced usage, you can pass Google’s native thinking config through `GoogleModelSettings.google_thinking_config`:

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
model = GoogleModel('gemini-3.5-flash')
model_settings = GoogleModelSettings(google_thinking_config={'include_thoughts': True, 'thinking_level': 'MEDIUM'})
agent = Agent(model, model_settings=model_settings)
...
```
See Thinking for the unified API and Gemini API docs for Google’s native thinking configuration.

You can customize the safety settings by setting the `google_safety_settings` field.

```
from google.genai.types import HarmBlockThreshold, HarmCategory
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
model_settings = GoogleModelSettings(
    google_safety_settings=[
        {
            'category': HarmCategory.HARM_CATEGORY_HATE_SPEECH,
            'threshold': HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
        }
    ]
)
model = GoogleModel('gemini-3-flash-preview')
agent = Agent(model, model_settings=model_settings)
...
```
See the Gemini API docs for more on safety settings.

You can return logprobs from the model in your response by setting `google_logprobs` and `google_top_logprobs` in the `GoogleModelSettings`.

This feature is only supported for non-streaming requests and Google Cloud.

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
from pydantic_ai.providers.google_cloud import GoogleCloudProvider
model_settings = GoogleModelSettings(
    google_logprobs=True, google_top_logprobs=2,
)
model = GoogleModel(
    model_name='gemini-2.5-flash',
    provider=GoogleCloudProvider(location='europe-west1'),
)
agent = Agent(model, model_settings=model_settings)
result = agent.run_sync('Your prompt here')
# Access logprobs from provider_details
logprobs = result.response.provider_details.get('logprobs')
avg_logprobs = result.response.provider_details.get('avg_logprobs')
```
See the Google Dev Blog for more information.

When you’ve created a Gemini cached content resource, pass its resource name through `google_cached_content` to reuse it across requests:

```
from pydantic_ai import Agent
from pydantic_ai.models.google import GoogleModel, GoogleModelSettings
model_settings = GoogleModelSettings(
    google_cached_content='projects/p/locations/global/cachedContents/your-cache-id',
)
agent = Agent(GoogleModel('gemini-2.5-pro'), model_settings=model_settings)
...
```
## Create a cached content resource

Pydantic AI doesn’t wrap the cache-management API — create the resource with the underlying google-genai SDK, then pass its name through `google_cached_content`:

```
from google.genai.types import Content, CreateCachedContentConfig, Part
from pydantic_ai.providers.google import GoogleProvider
provider = GoogleProvider(api_key='your-api-key')
cache = provider.client.caches.create(
    model='gemini-2.5-flash',
    config=CreateCachedContentConfig(
        system_instruction='You are a geography expert. Be concise.',
        contents=[Content(role='user', parts=[Part(text='...long context to cache...')])],
        ttl='3600s',
    ),
)
print(cache.name)
#> cachedContents/abc123...
```
Caches have a minimum size (≈1024 tokens for `gemini-2.5-flash`, ≈4096 for `gemini-2.5-pro`) and a TTL — see the Gemini caching docs for the current thresholds, pricing, and `list` / `update` / `delete` operations.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/google
