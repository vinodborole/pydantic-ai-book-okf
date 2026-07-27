---
type: Web Page
title: Process Event Stream | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/process-event-stream
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Process Event Stream

[ ProcessEventStream](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessEventStream) is a 

[capability](/docs/ai/capabilities/overview)that forwards the agent’s stream of

[s — model streaming and tool execution events — to a handler. When it’s registered,](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)

`AgentStreamEvent``agent.run()` automatically enables streaming, so the handler fires without passing an explicit [argument:](/docs/ai/core-concepts/agent#streaming-all-events)

`event_stream_handler`For example, forward events to a websocket, progress bar, or audit log.

The handler comes in two forms:

- An `EventStreamHandler``async def`returning`None`, as above. Events are forwarded to the handler and passed through unchanged, so multiple handlers (and a top-level`event_stream_handler`argument) can all observe the same stream. Events are delivered synchronously, so a slow handler back-pressures the rest of the stream.
- An `EventStreamProcessor`— an async generator that yields events. What it yields replaces the stream for downstream consumers, so it can modify, drop, or add events.

Registering the capability composes with other streaming mechanisms: see [Streaming all events](/docs/ai/core-concepts/agent#streaming-all-events) for the event vocabulary and handler examples.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/process-event-stream
