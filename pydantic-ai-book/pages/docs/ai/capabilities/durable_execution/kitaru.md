---
type: Web Page
title: Kitaru | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/durable_execution/kitaru
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Kitaru

[Kitaru](https://docs.zenml.io/kitaru) is a durable execution layer for AI agents. Its Pydantic AI adapter is provided by the `kitaru` package through `kitaru.adapters.pydantic_ai`, rather than by `pydantic_ai.durable_exec`.

Kitaru records agent progress as **flows** and **checkpoints**. A flow is the durable run you can resume later. A checkpoint is a completed model request, tool call, MCP invocation, or human wait that Kitaru can reuse during recovery.

For example, imagine an agent calls a model, gets a useful response, starts a tool call, and then the process crashes. Without durable execution, restarting the program usually repeats the model request and may repeat later side effects too. With Kitaru, the restarted flow can replay the run, reuse the completed checkpoint for the model request, and continue from the first incomplete point.

This is useful for long-running agents, human-in-the-loop workflows, and applications where a repeated model request or external API call would cost money, take time, or duplicate a side effect.

When you call a `KitaruAgent`, your application still calls the underlying Pydantic AI agent. Kitaru starts or resumes a flow for that call. With the default `"calls"` checkpoint strategy, it records completed model requests, tool calls, MCP invocations, and human waits in Kitaru’s checkpoint storage. On recovery, Kitaru runs the Python function again until it reaches an operation that already has a checkpoint, returns the saved result for that operation, and then continues from the first operation that has not completed.

You can make a normal [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) durable by wrapping it with `KitaruAgent` from `kitaru.adapters.pydantic_ai`. Install Kitaru separately from Pydantic AI:

For local development with Kitaru’s local server, install the `local` extra too:

Initialize the project and check your connection before running the examples:

Here is the smallest durable Pydantic AI agent using Kitaru:

`KitaruAgent` does not replace the original agent object. With the default call-level checkpoint strategy, it delegates to that agent for the actual Pydantic AI run and records recoverable operations while the run is executing:

- model requests;
- Pydantic AI tool calls;
- MCP tool calls;
- `@hitl_tool` human waits.

It exposes the usual run methods, including `Agent.run` and `Agent.run_sync`. The original [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) can still be used normally outside Kitaru.

Use the short wrapper for first experiments. Use an explicit flow when the run needs to move beyond your local process.

`KitaruAgent` supports two checkpoint strategies:

| Strategy | Default? | What gets persisted | Best for | 
|---|---|---|---|
| `"calls"` | Yes | Replay-safe model requests, tool calls, MCP invocations, and human waits are persisted as separate checkpoints. | Most agents, especially when individual calls are expensive or have side effects. | 
| `"turn"` | No | One checkpoint wraps the full agent run. | Simpler runs where per-call checkpoints are unnecessary, or cases where streaming constraints require a full-turn checkpoint. | 

With the default `"calls"` strategy, Kitaru cannot create nested checkpoints inside a user-defined `@kitaru.checkpoint` body. If `durable_agent.run_sync(...)` runs inside a user `@kitaru.checkpoint`, Kitaru records the whole agent turn under that outer checkpoint instead of creating separate model, tool, or MCP checkpoint rows.

See the [Kitaru Pydantic AI adapter guide](https://docs.zenml.io/kitaru/adapters/pydantic-ai) for advanced checkpoint configuration.

For pure human approval or data-entry gates, prefer Kitaru’s `@hitl_tool`. It turns the wait into a durable tool call: the process can stop while waiting for a human response, and recovery can continue after the response is available.

```
from kitaru.adapters.pydantic_ai import hitl_tool
@hitl_tool(question='Approve publishing this answer?', schema=bool)
def approve_publish(summary: str) -> bool: ...
```
For Pydantic AI’s own deferred tool patterns, see [deferred tools](/docs/ai/tools-toolsets/deferred-tools).

Kitaru supports Pydantic AI streaming with some constraints. For event streaming, prefer the [`event_stream_handler`](/docs/ai/core-concepts/agent#streaming-all-events) argument on `Agent.run`. When a run uses `event_stream_handler`, Kitaru falls back to a turn checkpoint for that call.

If you use [`run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) or [`iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.iter), wrap the streaming call in an explicit `@kitaru.checkpoint`. This gives Kitaru one durable operation to replay instead of trying to persist each streamed event separately.

When using Kitaru with Pydantic AI:

- Define the agent with a concrete model at construction time, such as `Agent('openai:gpt-5-nano', ...)` .
- Give each durable agent a stable `name` ; Kitaru uses it to identify persisted work across runs.
- Do not override the model per run with `model=` when using`KitaruAgent` .
- Use an explicit `@kitaru.flow` for remote stacks and production services; automatic flow creation is local-only.
- Avoid nested Kitaru checkpoints inside user-defined `@kitaru.checkpoint` bodies.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/durable_execution/kitaru
