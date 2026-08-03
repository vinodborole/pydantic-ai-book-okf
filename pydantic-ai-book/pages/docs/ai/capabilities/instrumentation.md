---
type: Web Page
title: Instrumentation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/instrumentation
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Instrumentation

[`Instrumentation`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Instrumentation) is a [capability](/docs/ai/capabilities/overview) that instruments agent runs with OpenTelemetry tracing: it creates spans for the run itself, each model request, and each tool execution, following the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/). Combined with [Pydantic Logfire](/docs/ai/integrations/logfire) (or any OTel backend), it gives you full visibility into what your agent is doing:

Sets the global `TracerProvider` that `Instrumentation` uses by default. Any OpenTelemetry SDK configuration works too.

Pass [`InstrumentationSettings`](/docs/ai/api/models/instrumented/#pydantic_ai.models.instrumented.InstrumentationSettings) via `Instrumentation(settings=...)` to customize providers, content capture, and the conventions version. To instrument every agent in your application instead of attaching the capability per agent, use [`Agent.instrument_all()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.instrument_all).

Other capabilities can attach attributes to the created spans through the OpenTelemetry API (`opentelemetry.trace.get_current_span().set_attribute(...)`).

See [Debugging and Monitoring](/docs/ai/integrations/logfire) for the full guide: setup, what gets captured, semantic-conventions versions, and excluding sensitive or binary content.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/instrumentation
