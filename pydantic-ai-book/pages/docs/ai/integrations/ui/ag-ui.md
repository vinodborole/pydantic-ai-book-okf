---
type: Web Page
title: AG-UI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/ui/ag-ui
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# AG-UI

The Agent-User Interaction (AG-UI) Protocol is an open standard introduced by the CopilotKit team that standardises how frontend applications communicate with AI agents, with support for streaming, frontend tools, shared state, and custom events.

The only dependencies are:

- ag-ui-protocol: to provide the AG-UI types and encoder.
- starlette: to handle ASGI requests from a framework like FastAPI.

You can install Pydantic AI with the `ag-ui` extra to ensure you have all the
required AG-UI dependencies:

To run the examples you’ll also need:

- uvicorn or another ASGI compatible server

There are three ways to run a Pydantic AI agent based on AG-UI run input with streamed AG-UI events as output, from most to least flexible. If you’re using a Starlette-based web framework like FastAPI, you’ll typically want to use the second method.

- The `AGUIAdapter.run_stream()`method, when called on an`AGUIAdapter`instantiated with an agent and an AG-UI`RunAgentInput`object, will run the agent and return a stream of AG-UI events. It also takes optional`Agent.iter()`arguments including`deps`. Use this if you’re using a web framework not based on Starlette (e.g. Django or Flask) or want to modify the input or output some way.
- The `AGUIAdapter.dispatch_request()`class method takes an agent and a Starlette request (e.g. from FastAPI) coming from an AG-UI frontend, and returns a streaming Starlette response of AG-UI events that you can return directly from your endpoint. It also takes optional`Agent.iter()`arguments including`deps`, that you can vary for each request (e.g. based on the authenticated user). This is a convenience method that combines`AGUIAdapter.from_request()`,`AGUIAdapter.run_stream()`, and`AGUIAdapter.streaming_response()`.
- Build a stand-alone `Starlette`app with a single`/`route that calls`AGUIAdapter.dispatch_request()`. The same Starlette app can be mounted at a path in an existing FastAPI app.

This example uses `AGUIAdapter.run_stream()` and performs its own request parsing and response generation.
This can be modified to work with any web framework.

- `AGUIAdapter.build_run_input()`takes the request body as bytes and returns an AG-UI- `RunAgentInput`object. You can also use the- `AGUIAdapter.from_request()`class method to build an adapter directly from a request.
- `AGUIAdapter.run_stream()`runs the agent and returns a stream of AG-UI events. It supports the same optional arguments as- `Agent.run_stream_events()`, including- `deps`. You can also use- `AGUIAdapter.run_stream_native()`to run the agent and return a stream of Pydantic AI events instead, which can then be transformed into AG-UI events using- `AGUIAdapter.transform_stream()`.
- `AGUIAdapter.encode_stream()`encodes the stream of AG-UI events as strings according to the accept header value. You can also use- `AGUIAdapter.streaming_response()`to generate a streaming response directly from the AG-UI event stream returned by- `run_stream()`.

Since `app` is an ASGI application, it can be used with any ASGI server:

This will expose the agent as an AG-UI server, and your frontend can start sending requests to it.

This example uses `AGUIAdapter.dispatch_request()` to directly handle a FastAPI request and return a response. Something analogous to this will work with any Starlette-based web framework.

- This method essentially does the same as the previous example, but it’s more convenient to use when you’re already using a Starlette/FastAPI app.

Since `app` is an ASGI application, it can be used with any ASGI server:

This will expose the agent as an AG-UI server, and your frontend can start sending requests to it.

When you don’t already have a Starlette/FastAPI app to mount the route on, build a minimal `Starlette` app whose single `/` route calls `AGUIAdapter.dispatch_request()`:

Since `app` is an ASGI application, it can be used with any ASGI server:

This will expose the agent as an AG-UI server, and your frontend can start sending requests to it.

The Pydantic AI AG-UI integration supports all features of the spec:

The integration receives messages in the form of a
`RunAgentInput` object
that describes the details of the requested agent run including message history, state, and available tools.

These are converted to Pydantic AI types and passed to the agent’s run method. Events from the agent, including tool calls, are converted to AG-UI events and streamed back to the caller as Server-Sent Events (SSE).

A user request may require multiple round trips between client UI and Pydantic AI server, depending on the tools and events needed.

The integration provides full support for AG-UI state management, which enables real-time synchronization between agents and frontend applications.

In the example below we have document state which is shared between the UI and
server using the `StateDeps` dependencies type that can be used to automatically
validate state contained in `RunAgentInput.state` using a Pydantic `BaseModel` specified as a generic parameter.

Since `app` is an ASGI application, it can be used with any ASGI server:

AG-UI frontend tools are seamlessly provided to the Pydantic AI agent, enabling rich user experiences with frontend user interfaces.

Tools declared with `requires_approval=True` map onto AG-UI’s interrupt-aware run lifecycle. When the model proposes such a call, the run pauses and the adapter ends the SSE stream with a `RUN_FINISHED` event whose `outcome.type` is `"interrupt"` and whose `outcome.interrupts[]` describes each pending approval. The client renders an approval UI from that list and POSTs the next `RunAgentInput` with a `resume[]` array of `ResumeEntry` items addressing each interrupt.

The mapping the adapter applies (matching the AG-UI Python SDK field names):

| AG-UI direction | Pydantic AI source / sink | 
|---|---|
| `Interrupt.reason` | Always `"tool_call"`for`requires_approval=True`tools | 
| `Interrupt.tool_call_id` | The `ToolCallPart.tool_call_id`of the proposed call | 
| `Interrupt.id` | `f"int-{tool_call_id}"`(round-trips back to`tool_call_id`on resume) | 
| `Interrupt.metadata` | `DeferredToolRequests.metadata.get(tool_call_id)` | 
| `ResumeEntry.payload` | `{ "approved": bool, "editedArgs"?: object, "reason"?: string }` | 
| `payload.approved=True` | `ToolApproved` | 
| `payload.editedArgs` | `ToolApproved.override_args`(fully replaces the proposed args) | 
| `payload.approved=False` | `ToolDenied`with`message=payload.reason` | 
| `status="cancelled"` | `ToolDenied`with`message="Cancelled by user."`regardless of payload | 

The agent must include `DeferredToolRequests` in its `output_type` so the run can pause cleanly instead of erroring on the proposed call:

On the resumed turn the agent re-executes the tool against the **original** `tool_call_id`, so only a `TOOL_CALL_RESULT` event is emitted for that id — no fresh `TOOL_CALL_START`. This preserves the audit trail the AG-UI spec requires.

See Deferred tools and human-in-the-loop tool approval for the underlying Pydantic AI primitive that also works outside AG-UI.

Pydantic AI tools can send AG-UI events simply by returning a
`ToolReturn` object with a
`BaseEvent` (or a list of events) as `metadata`,
which allows for custom events and state updates.

Since `app` is an ASGI application, it can be used with any ASGI server:

AG-UI’s `RunAgentInput.messages` is fully client-controlled. The `AGUIAdapter` applies defaults to strip untrusted parts before the agent runs — see Trust model for client-submitted messages in the UI adapter overview, which covers system prompts, file URL schemes, uploaded files (`allow_uploaded_files`), and unresolved tool calls.

AG-UI has no native representation for agent-generated files (`FilePart`) or `UploadedFile` references, so they are omitted from `dump_messages` output by default. Set `AGUIAdapter.preserve_file_data` to `True` to round-trip them through reserved `pydantic_ai_*` activity messages, which a frontend completes by echoing those activity messages back on the next request. This is a representation opt-in, not a security one: an `UploadedFile` reconstructed from a round-tripped activity message is still subject to the inbound `allow_uploaded_files` gate before it reaches the agent.

Pydantic AI supports two ways to provide guidance to the model: `system_prompt` (stored in the message history as `SystemPromptPart`s) and `instructions` (injected fresh on every request, never persisted). When you control the server side, `instructions` is the recommended default.

The rest of this section only matters if you use `system_prompt`. If you only use `instructions`, there’s nothing to configure — they’re always applied regardless of the AG-UI message history.

For `system_prompt`, you choose who owns it with the `manage_system_prompt` parameter on `AGUIAdapter`:

- `'server'`(default): the agent’s configured- `system_prompt`is authoritative. Any- `SystemMessage`sent by the frontend is stripped with a warning (a malicious client could otherwise inject arbitrary instructions via crafted API requests), and the agent’s own system prompt is reinjected at the head of the first request via the- `ReinjectSystemPrompt`capability.
- `'client'`: the frontend owns the system prompt. Frontend- `SystemMessage`s are preserved as-is, and the agent’s configured- `system_prompt`is not injected — the caller is fully responsible for sending it on every turn if desired. To opt into fallback-to-configured behavior, add the- `ReinjectSystemPrompt`capability to your agent.

For more examples see
`pydantic_ai_examples.ag_ui`,
which includes a server for use with the
AG-UI Dojo.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/ui/ag-ui
