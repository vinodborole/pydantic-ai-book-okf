---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/overview
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Overview

Pydantic AI is model-agnostic and has built-in support for multiple model providers:

- [OpenAI](/docs/ai/models/openai)
- [Anthropic](/docs/ai/models/anthropic)
- [Gemini](/docs/ai/models/google) (via two different APIs: Gemini API and Google Cloud, formerly known as Vertex AI)
- [xAI](/docs/ai/models/xai)
- [Bedrock](/docs/ai/models/bedrock)
- [Cerebras](/docs/ai/models/cerebras)
- [Cohere](/docs/ai/models/cohere)
- [Groq](/docs/ai/models/groq)
- [Hugging Face](/docs/ai/models/huggingface)
- [Mistral](/docs/ai/models/mistral)
- [OpenRouter](/docs/ai/models/openrouter)
- [Z.AI](/docs/ai/models/zai)

In addition, many providers are compatible with the OpenAI API, and can be used with `OpenAIChatModel` in Pydantic AI:

Pydantic AI also comes with [`TestModel`](/docs/ai/api/models/test) and [`FunctionModel`](/docs/ai/api/models/function)
for testing and development.

To use each model provider, you need to configure your local environment and make sure you have the right packages installed. If you try to use the model without having done so, you’ll be told what to install.

Pydantic AI uses a few key terms to describe how it interacts with different LLMs:

- **Model** : This refers to the Pydantic AI class used to make requests following a specific LLM API
(generally by wrapping a vendor-provided SDK, like the`openai` python SDK). These classes implement a
vendor-SDK-agnostic API, ensuring a single Pydantic AI agent is portable to different LLM vendors without
any other code changes just by swapping out the Model it uses. Model classes are named
roughly in the format`<VendorSdk>Model` , for example, we have`OpenAIChatModel` ,`AnthropicModel` ,`GoogleModel` ,
etc. When using a Model class, you specify the actual LLM model name (e.g.,`gpt-5` ,`claude-sonnet-4-5` ,`gemini-3-flash-preview` ) as a parameter.
- **Provider** : This refers to provider-specific classes which handle the authentication and connections
to an LLM vendor. Passing a non-default*Provider* as a parameter to a Model is how you can ensure
that your agent will make requests to a specific endpoint, or make use of a specific approach to
authentication (e.g., you can use Azure auth with the`OpenAIChatModel` by way of the`AzureProvider` ).
In particular, this is how you can make use of an AI gateway, or an LLM vendor that offers API compatibility
with the vendor SDK used by an existing Model (such as`OpenAIChatModel` ).
- **Profile** : This refers to a description of how requests to a specific model or family of models need to be
constructed to get the best results, independent of the model and provider classes used.
For example, different models have different restrictions on the JSON schemas that can be used for tools,
and the same schema transformer needs to be used for Gemini models whether you’re using`GoogleModel` with model name`gemini-3-pro-preview` , or`OpenAIChatModel` with`OpenRouterProvider` and model name`google/gemini-3-pro-preview` .

When you instantiate an [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) with just a name formatted as `<provider>:<model>`, e.g. `openai:gpt-5.2` or `openrouter:google/gemini-3-pro-preview`,
Pydantic AI will automatically select the appropriate model class, provider, and profile.
If you want to use a different provider or profile, you can instantiate a model class directly and pass in `provider` and/or `profile` arguments.

A model’s [`ModelProfile`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile) also describes what the model can do. It is a `TypedDict`, so you read capability flags with normal dictionary access via `model.profile` — for example [`supports_tools`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.supports_tools), [`supports_json_schema_output`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.supports_json_schema_output), and [`supported_native_tools`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.supported_native_tools). This is useful when you want to branch on a capability rather than discover a limitation at request time — for example checking whether a model supports tool calling, native JSON-schema output, or a specific native tool before relying on it:

```
from pydantic_ai.models.test import TestModel
from pydantic_ai.native_tools import WebSearchTool
model = TestModel()
profile = model.profile
print(profile['supports_tools'])
#> True
print(profile['supports_json_schema_output'])
#> False
print(WebSearchTool in profile['supported_native_tools'])
#> True
```
`model.profile` is usually the fully *resolved* profile: keys from [`DEFAULT_PROFILE`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.DEFAULT_PROFILE) are merged with the provider’s defaults, so direct key access like `profile['supports_tools']` works. If you supply `profile=` as a callable (or otherwise have a partial profile dict), use `profile.get('supports_tools', DEFAULT_PROFILE['supports_tools'])` (after importing `DEFAULT_PROFILE`) to tolerate missing keys.
Any [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) instance exposes its resolved profile the same way, so the same check works whether the model was selected automatically from a `<provider>:<model>` name or instantiated directly. Don’t confuse this with [Capabilities](/docs/ai/capabilities/overview), which are reusable bundles of tools, hooks, and settings you add to an agent — the profile describes what the underlying model itself supports.

When a [`Provider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.Provider) creates its own HTTP client (i.e. you don’t pass a custom `http_client`), it owns that client’s lifecycle. Using the [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) as an async context manager ensures the HTTP client is closed cleanly on exit:

```
from pydantic_ai import Agent
agent = Agent('openai:gpt-5.2')
async def main():
    async with agent:
        result = await agent.run('What is the capital of France?')
        print(result.output)
        #> The capital of France is Paris.
```
You can also use a [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) or [`Provider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.Provider) directly as an async context manager for the same effect.

If you provide your own `http_client`, you are responsible for closing it yourself.

To implement support for a model API that’s not already supported, you will need to subclass the [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) abstract base class.
For streaming, you’ll also need to implement the [`StreamedResponse`](/docs/ai/api/models/base/#pydantic_ai.models.StreamedResponse) abstract base class.

The best place to start is to review the source code for existing implementations, e.g. [`OpenAIChatModel`](https://github.com/pydantic/pydantic-ai/blob/main/pydantic_ai_slim/pydantic_ai/models/openai.py).

For details on when we’ll accept contributions adding new models to Pydantic AI, see the [contributing guidelines](/docs/ai/project/contributing#new-model-rules).

You can limit the number of concurrent HTTP requests to a model using the
[`ConcurrencyLimitedModel`](/docs/ai/api/pydantic-ai/concurrency/#pydantic_ai.ConcurrencyLimitedModel) wrapper.
This is useful for respecting rate limits or managing resource usage when running many agents in parallel.

The `limiter` parameter accepts:

- An integer for simple limiting (e.g., `limiter=5` )
- A [`ConcurrencyLimit`](/docs/ai/api/pydantic-ai/concurrency/#pydantic_ai.ConcurrencyLimit) for advanced configuration with backpressure control
- A [`ConcurrencyLimiter`](/docs/ai/api/pydantic-ai/concurrency/#pydantic_ai.ConcurrencyLimiter) for sharing limits across multiple models

To share a concurrency limit across multiple models (e.g., different models from the same provider),
you can create a [`ConcurrencyLimiter`](/docs/ai/api/pydantic-ai/concurrency/#pydantic_ai.ConcurrencyLimiter) and pass it to
multiple `ConcurrencyLimitedModel` instances:

When instrumentation is enabled, requests waiting for a concurrency slot appear as spans with
attributes showing the queue depth and configured limits. The `name` parameter on
`ConcurrencyLimiter` helps identify shared limiters in traces.

When a provider returns a 4xx or 5xx response, Pydantic AI raises a
[`ModelHTTPError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError). The exception exposes the
[`status_code`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError.status_code), the response
[`body`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError.body), and the provider’s
**response headers** via the [`headers`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError.headers)
attribute (a `dict[str, str]` with lowercase keys, or `None` for providers that don’t
surface headers, such as gRPC-based providers).

The motivating use case is propagating the `Retry-After` header from a 429 response to a
caller’s own HTTP client.  A convenience property
[`retry_after`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError.retry_after) parses that header and
returns the number of seconds to wait as a `float`, handling both the integer
delta-seconds and HTTP-date formats:

You can use [`FallbackModel`](/docs/ai/api/models/fallback/#pydantic_ai.models.fallback.FallbackModel) to attempt multiple models
in sequence until one succeeds. Pydantic AI can switch to the next model when the current model
raises an exception (like a 4xx/5xx API error) **or** when the response content indicates a semantic
failure (like a truncated response or a failed native tool call).

By default, fallback triggers on [`ModelAPIError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelAPIError) (4xx/5xx API errors),
so you don’t need to configure anything for the most common use case.

This behavior is controlled by the `fallback_on` parameter (see
[`FallbackModel`](/docs/ai/api/models/fallback/#pydantic_ai.models.fallback.FallbackModel)), which accepts exception types,
exception handlers, and response handlers — all of which can be sync or async.

In the following example, the agent first makes a request to the OpenAI model (which fails due to an invalid API key), and then falls back to the Anthropic model.

The `ModelResponse` message above indicates in the `model_name` field that the output was returned by the Anthropic model, which is the second model specified in the `FallbackModel`.

You can configure different [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) for each model in a fallback chain by passing the `settings` parameter when creating each model. This is particularly useful when different providers have different optimal configurations:

In this example, if the OpenAI model fails, the agent will automatically fall back to the Anthropic model with its own configured settings. The `FallbackModel` itself doesn’t have settings - it uses the individual settings of whichever model successfully handles the request.

The next example demonstrates the exception-handling capabilities of `FallbackModel`.
If all models fail, a [`FallbackExceptionGroup`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.FallbackExceptionGroup) is raised, which
contains all the exceptions encountered during the `run` execution.

Since [`except*`](https://docs.python.org/3/reference/compound_stmts.html#except-star) is only supported
in Python 3.11+, we use the [`exceptiongroup`](https://github.com/agronholm/exceptiongroup) backport
package for earlier Python versions:

By default, the `FallbackModel` only moves on to the next model if the current model raises a
[`ModelAPIError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelAPIError), which includes
[`ModelHTTPError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelHTTPError). You can customize this behavior by
passing a custom `fallback_on` argument to the `FallbackModel` constructor.

In addition to exception-based fallback, you can also trigger fallback based on the **content** of a model’s response. This is useful when a model returns a successful HTTP response (no exception), but the response content indicates a semantic failure — for example, an unexpected finish reason or a native tool reporting failure.

The `fallback_on` parameter accepts:

- A tuple of exception types: `(ModelAPIError, ModelHTTPError)`
- An exception handler (sync or async): `lambda exc: isinstance(exc, MyError)`
- A response handler (sync or async): `def check(r: ModelResponse) -> bool`
- A list mixing all of the above: `[ModelAPIError, exc_handler, response_handler]`

Handler type is auto-detected by inspecting type hints on the first parameter. If the first parameter is hinted as [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse), it’s a response handler. Otherwise (including untyped handlers and lambdas), it’s an exception handler.

As the hints are resolved at runtime, every annotated type in the handler signature must be imported at runtime rather than only under `if TYPE_CHECKING:`. If any annotation can’t be resolved, a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) is raised instead of the handler being silently treated as an exception handler.

A simple use case is checking the model’s finish reason — for example, falling back if the response was truncated due to length limits:

A more complex use case is when using native tools like web search or URL fetching. For example, Google’s [`WebFetchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebFetchTool) may return a successful response with a status indicating the URL fetch failed:

Response handlers receive the [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) returned by the model and should return `True` to trigger fallback to the next model, or `False` to accept the response.

You can combine exception types, exception handlers, and response handlers in a single list:

When using `FallbackModel`, it’s important to understand that [`FallbackExceptionGroup`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.FallbackExceptionGroup)
inherits from Python’s [`ExceptionGroup`](https://docs.python.org/3/library/exceptions.html#ExceptionGroup). This means
that existing exception handling code that catches specific exceptions (like `ModelAPIError`) won’t automatically catch
the individual exceptions wrapped inside the group.

For example, if you have middleware or a decorator that catches `ModelAPIError`:

This decorator will miss `ModelAPIError` exceptions when using `FallbackModel`, because they’re wrapped in a
`FallbackExceptionGroup` containing one exception per failed model, in the order the models were tried.

To handle both cases, you can use Python 3.11+ `except*` syntax, which catches matching exceptions from
exception groups as well as bare exceptions. Note that `except*` always delivers the caught exceptions as an
`ExceptionGroup` (even if the original was a bare exception), so re-raising will propagate an `ExceptionGroup`
rather than the original exception type:

You can also catch `FallbackExceptionGroup` directly if you want to handle it specifically:

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/overview
