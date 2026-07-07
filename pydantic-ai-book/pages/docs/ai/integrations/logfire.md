---
type: Web Page
title: Debugging & Monitoring with Pydantic Logfire | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/logfire
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Debugging & Monitoring with Pydantic Logfire

Applications that use LLMs have some challenges that are well known and understood: LLMs are **slow**, **unreliable** and **expensive**.

These applications also have some challenges that most developers have encountered much less often: LLMs are **fickle** and **non-deterministic**. Subtle changes in a prompt can completely change a model’s performance, and there’s no `EXPLAIN` query you can run to understand why.

To build successful applications with LLMs, we need new tools to understand both model performance, and the behavior of applications that rely on them.

LLM Observability tools that just let you understand how your model is performing are useless: making API calls to an LLM is easy, it’s building that into an application that’s hard.

Pydantic Logfire is an observability platform developed by the team who created and maintain Pydantic Validation and Pydantic AI. Logfire aims to let you understand your entire application: Gen AI, classic predictive AI, HTTP traffic, database queries and everything else a modern application needs, all using OpenTelemetry.

Pydantic AI has built-in (but optional) support for Logfire. That means if the `logfire` package is installed and configured and agent instrumentation is enabled then detailed information about agent runs is sent to Logfire. Otherwise there’s virtually no overhead and nothing is sent.

Here’s an example showing details of running the Weather Agent in Logfire:

A trace is generated for the agent run, and spans are emitted for each model request and tool call.

To use Logfire, you’ll need a Logfire account. The Logfire Python SDK is included with `pydantic-ai`:

Or if you’re using the slim package, you can install it with the `logfire` optional group:

Then authenticate your local environment with Logfire:

And configure a project to send data to:

(Or use an existing project with `logfire projects use`)

This will write to a `.logfire` directory in the current working directory, which the Logfire SDK will use for configuration at run time.

With that, you can start using Logfire to instrument Pydantic AI code:

`logfire.configure()` configures the SDK, by default it will find the write token from the `.logfire` directory, but you can also pass a token directly.

`logfire.instrument_pydantic_ai()` enables instrumentation of Pydantic AI.

Since we've enabled instrumentation, a trace will be generated for each run, with spans emitted for models calls and tool function execution

Passing `name` is optional but recommended: it labels the agent's run span in Logfire. When omitted, the name is inferred from the variable the agent is assigned to and falls back to `'agent'` when it can't be (e.g. agents kept in a list or dict). This matters most when several agents run in one app and you need to tell their traces apart.

*(This example is complete, it can be run “as is”)*

Which will display in Logfire thus:

The Logfire documentation has more details on how to use Logfire, including how to instrument other libraries like HTTPX and FastAPI.

Since Logfire is built on OpenTelemetry, you can use the Logfire Python SDK to send data to any OpenTelemetry collector, see below.

To demonstrate how Logfire can let you visualise the flow of a Pydantic AI run, here’s the view you get from Logfire while running the chat app examples:

We can also query data with SQL in Logfire to monitor the performance of an application. Here’s a real world example of using Logfire to monitor Pydantic AI runs inside Logfire itself:

As per Hamel Husain’s influential 2024 blog post “Fuck You, Show Me The Prompt.” (bear with the capitalization, the point is valid), it’s often useful to be able to view the raw HTTP requests and responses made to model providers.

To observe raw HTTP requests made to model providers, you can use Logfire’s HTTPX instrumentation since all provider SDKs (except for Bedrock) use the HTTPX library internally:

See the `logfire.instrument_httpx` docs more details, `capture_all=True` means both headers and body are captured for both the request and response.

Pydantic AI’s instrumentation uses OpenTelemetry (OTel), which Logfire is based on.

This means you can debug and monitor Pydantic AI with any OpenTelemetry backend.

Pydantic AI follows the OpenTelemetry Semantic Conventions for Generative AI systems, so while we think you’ll have the best experience using the Logfire platform 😉, you should be able to use any OTel service with GenAI support.

You can use the Logfire SDK completely freely and send the data to any OpenTelemetry backend.

Here’s an example of configuring the Logfire library to send data to the excellent otel-tui — an open source terminal based OTel backend and viewer (no association with Pydantic Validation).

Run `otel-tui` with docker (see the otel-tui readme for more instructions):

then run,

Set the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to the URL of your OpenTelemetry backend. If you're using a backend that requires authentication, you may need to set other environment variables. Of course, these can also be set outside the process, e.g. with `export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`.

We configure Logfire to disable sending data to the Logfire OTel backend itself. If you removed `send_to_logfire=False`, data would be sent to both Logfire and your OpenTelemetry backend.

Running the above code will send tracing data to `otel-tui`, which will display like this:

Running the weather agent example connected to `otel-tui` shows how it can be used to visualise a more complex trace:

For more information on using the Logfire SDK to send data to alternative backends, see the Logfire documentation.

You can also emit OpenTelemetry data from Pydantic AI without using Logfire at all.

To do this, you’ll need to install and configure the OpenTelemetry packages you need. To run the following examples, use

Because Pydantic AI uses OpenTelemetry for observability, you can easily configure it to send data to any OpenTelemetry-compatible backend, not just our observability platform Pydantic Logfire.

The following providers have dedicated documentation on Pydantic AI:

- Langfuse
- W&B Weave
- Arize
- Openlayer
- LangWatch
- Patronus AI
- Opik
- mlflow
- Agenta
- Braintrust
- SigNoz
- Laminar
- Respan
- Raindrop

By default, model request spans use the standard `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` attributes, while agent run spans use `gen_ai.aggregated_usage.input_tokens`, `gen_ai.aggregated_usage.output_tokens`, and `gen_ai.aggregated_usage.details.*`.

This avoids double-counting in observability backends that aggregate usage attributes across parent and child spans, since agent run spans report the sum of their child model request spans’ usage.

If you want agent run spans to use the standard `gen_ai.usage.*` attributes and handle double-counting in your backend, disable aggregated usage attribute names:

```
from pydantic_ai import Agent
from pydantic_ai.models.instrumented import InstrumentationSettings
Agent.instrument_all(InstrumentationSettings(use_aggregated_usage_attribute_names=False))
```
Pydantic AI follows the OpenTelemetry Semantic Conventions for Generative AI systems, specifically version 1.37.0 of the conventions. The instrumentation format can be configured using the `version` parameter of `InstrumentationSettings`.

**The default is  version=5**.

Versions 2, 3, and 4 are deprecated compatibility formats. Passing one of these versions to `InstrumentationSettings` emits a `PydanticAIDeprecationWarning`; use version 5 unless you are temporarily preserving an older telemetry pipeline.

Uses the newer OpenTelemetry GenAI spec and stores messages in the following attributes:

- `gen_ai.system_instructions`for instructions passed to the agent
- `gen_ai.input.messages`and- `gen_ai.output.messages`on model request spans
- `pydantic_ai.all_messages`on agent run spans

Some span and attribute names are not fully spec-compliant for compatibility reasons. Use version 5 for current telemetry.

Builds on version 2 with the following improvements:

- **Spec-compliant span names:**- `agent run`becomes- `invoke_agent {gen_ai.agent.name}`(with the agent name filled in)
- `running tool`becomes- `execute_tool {gen_ai.tool.name}`(with the tool name filled in)
 
- **Spec-compliant attribute names:**- `tool_arguments`becomes- `gen_ai.tool.call.arguments`
- `tool_response`becomes- `gen_ai.tool.call.result`
 
- **Thinking tokens support:**Captures thinking/reasoning tokens when available

Builds on version 3 with improved multimodal content handling to better align with the GenAI semantic conventions for multimodal inputs:

**URL-based media (ImageUrl, AudioUrl, VideoUrl):**

- Old (v2-3): `{"type": "image-url", "url": "..."}`
- New (v4): `{"type": "uri", "modality": "image", "uri": "...", "mime_type": "..."}`

**Inline binary content (BinaryContent, FilePart):**

- Old (v2-3): `{"type": "binary", "media_type": "...", "content": "..."}`
- New (v4): `{"type": "blob", "modality": "image", "mime_type": "...", "content": "..."}`

Note: The `modality` field is only included for image, audio, and video content types as specified in the OTel spec. DocumentUrl and unsupported media types omit the `modality` field.

Builds on version 4 with improved handling of deferred tool calls:

- `CallDeferred`and- `ApprovalRequired`exceptions no longer record an exception event or set the span status to ERROR — the span is left as UNSET, since deferrals are control flow, not errors.

Note that the OpenTelemetry Semantic Conventions are still experimental and are likely to change.

By default, the global `TracerProvider` is used. This is set automatically by `logfire.configure()`. It can also be set by the `set_tracer_provider` function in the OpenTelemetry Python SDK. You can set custom providers with `InstrumentationSettings`.

For privacy and security reasons, you may want to monitor your agent’s behavior and performance without exposing sensitive user data or proprietary prompts in your observability platform. Pydantic AI allows you to exclude the actual content from telemetry while preserving the structural information needed for debugging and monitoring.

When `include_content=False` is set, Pydantic AI will exclude sensitive content from telemetry, including user prompts and model completions, tool call arguments and responses, and any other message content.

This setting is particularly useful in production environments where compliance requirements or data sensitivity concerns make it necessary to limit what content is sent to your observability platform.

Use the agent’s `metadata` parameter to attach additional data to the agent’s span.
When instrumentation is enabled, the computed metadata is recorded on the agent span under the `metadata` attribute.
See the usage and metadata example in the agents guide for details and usage.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/logfire
