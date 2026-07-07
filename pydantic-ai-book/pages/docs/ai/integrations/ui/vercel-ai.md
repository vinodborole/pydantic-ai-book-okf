---
type: Web Page
title: Vercel AI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/ui/vercel-ai
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Vercel AI

Pydantic AI natively supports the Vercel AI Data Stream Protocol to receive agent run input from, and stream events to, a frontend using AI SDK UI hooks like `useChat`. You can optionally use AI Elements for pre-built UI components.

The `VercelAIAdapter` class is responsible for transforming agent run input received from the frontend into arguments for `Agent.run_stream_events()`, running the agent, and then transforming Pydantic AI events into Vercel AI events. The event stream transformation is handled by the `VercelAIEventStream` class, but you typically won’t use this directly.

If you’re using a Starlette-based web framework like FastAPI, you can use the `VercelAIAdapter.dispatch_request()` class method from an endpoint function to directly handle a request and return a streaming response of Vercel AI events. This is demonstrated in the next section.

If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `VercelAIAdapter` instance and directly use its methods. This is demonstrated in “Advanced Usage” section below.

Besides the request, `VercelAIAdapter.dispatch_request()` takes the agent, the same optional arguments as `Agent.run_stream_events()`, and an optional `on_complete` callback function that receives the completed `AgentRunResult` and can optionally yield additional Vercel AI events.

If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `VercelAIAdapter` instance and directly use its methods, which can be chained to accomplish the same thing as the `VercelAIAdapter.dispatch_request()` class method shown above:

- The `VercelAIAdapter.build_run_input()`class method takes the request body as bytes and returns a Vercel AI`RequestData`run input object, which you can then pass to the`VercelAIAdapter()`constructor along with the agent.- You can also use the `VercelAIAdapter.from_request()`class method to build an adapter directly from a Starlette/FastAPI request.
 
- You can also use the 
- The `VercelAIAdapter.run_stream()`method runs the agent and returns a stream of Vercel AI events. It supports the same optional arguments as`Agent.run_stream_events()`and an optional`on_complete`callback function that receives the completed`AgentRunResult`and can optionally yield additional Vercel AI events.- You can also use `VercelAIAdapter.run_stream_native()`to run the agent and return a stream of Pydantic AI events instead, which can then be transformed into Vercel AI events using`VercelAIAdapter.transform_stream()`.
 
- You can also use 
- The `VercelAIAdapter.encode_stream()`method encodes the stream of Vercel AI events as SSE (HTTP Server-Sent Events) strings, which you can then return as a streaming response.- You can also use `VercelAIAdapter.streaming_response()`to generate a Starlette/FastAPI streaming response directly from the Vercel AI event stream returned by`run_stream()`.
 
- You can also use 

Pydantic AI tools can send Vercel AI data stream chunks by returning a
`ToolReturn` object with a data-carrying chunk
(or a list of chunks) as `metadata`.
The supported chunk types are `DataChunk`,
`SourceUrlChunk`,
`SourceDocumentChunk`,
and `FileChunk`.
This is useful for attaching structured data to the frontend alongside the tool result, such as source URLs or custom data payloads.

Vercel AI SDK client-side tools run in the browser and submit their result back to the server, where Pydantic AI resolves them as external tool calls. Such a tool can return a file by putting a shape in its output that matches one of Pydantic AI’s multimodal content types; Pydantic AI deserializes it into that type (via the same `ToolReturnContent` discriminator used for round-tripping) before the run continues. Use the type’s snake_case field names — these are validated as Pydantic AI models, not Vercel-cased payloads, so `media_type` deserializes but `mediaType` stays an opaque dict. Three shapes are supported:

- **Inline bytes**— a- `BinaryContent`shape,- `{ kind: 'binary', media_type: 'image/png', data: <bytes> }`(image media types become- `BinaryImage`). The- `data`field accepts a base64 string, or the raw byte shapes a JavaScript frontend produces when it forwards a- `Uint8Array`or Node- `Buffer`through- `JSON.stringify`without encoding it first (- `{ "0": 137, "1": 80, ... }`or- `{ "type": "Buffer", "data": [137, 80, ...] }`) — all normalized to bytes at the wire boundary, so a client-side tool can return binary data without base64-encoding it by hand.
- **A file URL**— a- `FileUrl`shape such as- `{ kind: 'image-url', url: 'https://...' }`or- `{ kind: 'document-url', url: 'https://...' }`. This is often more efficient than inlining the bytes, since only the reference crosses the wire and the provider fetches the file directly — a good fit when the file already lives at a URL the frontend trusts. The URL is honored only if its scheme passes the adapter’s- `allowed_file_url_schemes`allowlist (- `http`/- `https`by default); see the trust model.
- **A provider-hosted file**— an- `UploadedFile`shape,- `{ kind: 'uploaded-file', file_id: 'file-123', provider_name: 'openai' }`, referencing a file already uploaded to the provider’s storage. Honored only when- `allow_uploaded_files`is- `True`, since the server resolves it against the provider’s file API using its own credentials.

`VercelAIAdapter.dump_messages` writes `ModelRequest.metadata` and `ModelResponse.metadata` into Vercel AI `UIMessage.metadata`, and stores the message `timestamp` under a reserved `pydantic_ai` key so it survives the round-trip. `VercelAIAdapter.load_messages` restores it on the way back.

When streaming, the timestamp is also emitted as a Vercel AI `message-metadata` chunk after the final step, so frontends using AI SDK UI can persist it with the assistant message. Request-side messages have no analogous chunk — frontends rebuilding history purely from streamed chunks see timestamps only on assistant responses, whereas `dump_messages` populates both sides.

`UIMessage.metadata` is fully client-controlled, so only `timestamp` is round-tripped: server-side fields such as `usage`, `model_name`, and `provider_*` are deliberately excluded — dumping them could leak infrastructure details, and restoring them would trust client-submitted history for values the server owns. Broadening the round-trip behind an explicit user-controlled opt-in is tracked in issue #5174.

Vercel AI’s request `messages` array is fully client-controlled, and the protocol round-trips approval responses and tool results through the message history. The `VercelAIAdapter` applies defaults to strip untrusted parts before the agent runs — see Trust model for client-submitted messages in the UI adapter overview, which covers system prompts, file URL schemes, uploaded files (`allow_uploaded_files`), and unresolved tool calls.

Pydantic AI supports human-in-the-loop tool approval workflows with AI SDK UI, allowing users to approve or deny tool executions before they run. See the deferred tool calls documentation for details on setting up tools that require approval.

To enable tool approval streaming, pass `sdk_version=6` to `dispatch_request`:

```
@app.post('/chat')
async def chat(request: Request) -> Response:
    return await VercelAIAdapter.dispatch_request(request, agent=agent, sdk_version=6)
```
When `sdk_version=6`, the adapter will:

- Emit `tool-approval-request`chunks when tools with`requires_approval=True`are called
- Automatically extract approval responses from follow-up requests
- Emit `tool-output-denied`chunks for rejected tools

On the frontend, AI SDK UI’s `useChat` hook handles the approval flow. You can use the `Confirmation` component from AI Elements for a pre-built approval UI, or build your own using the hook’s `addToolApprovalResponse` function.

Tool approval responses are trusted from the request by design, matching the protocol’s round-trip through `useChat`’s `addToolApprovalResponse` and the reference Next.js backend. If your application needs the approval decision tied to server-side state rather than the request, intercept `DeferredToolRequests`, persist the approval IDs server-side, and pass explicit `deferred_tool_results` when resuming.

`tool-input-available` is emitted **after** the agent has validated the call against the tool’s schema and any custom `args_validator`, so the chunk only fires once the args are known to be acceptable. The chunk’s `input` field carries the raw arguments the model emitted.

When validation fails, the adapter emits `tool-input-error` instead of `tool-input-available`. The chunk carries the same `tool_call_id`, `tool_name`, and `input` (the raw arguments) plus an `error_text` field rendered from the retry prompt that will be sent back to the model. The agent will retry the call (subject to the tool’s `retries` setting) and emit a new `tool-input-(available|error)` for each attempt.

Pydantic AI supports two ways to provide guidance to the model: `system_prompt` (stored in the message history as `SystemPromptPart`s) and `instructions` (injected fresh on every request, never persisted). When you control the server side, `instructions` is the recommended default.

The rest of this section only matters if you use `system_prompt`. If you only use `instructions`, there’s nothing to configure — they’re always applied regardless of the frontend message history.

For `system_prompt`, you choose who owns it with the `manage_system_prompt` parameter on `VercelAIAdapter`:

- `'server'`(default): the agent’s configured- `system_prompt`is authoritative. Any system message sent by the frontend is stripped with a warning (a malicious client could otherwise inject arbitrary instructions via crafted API requests), and the agent’s own system prompt is reinjected at the head of the first request via the- `ReinjectSystemPrompt`capability.
- `'client'`: the frontend owns the system prompt. Frontend system messages are preserved as-is, and the agent’s configured- `system_prompt`is not injected — the caller is fully responsible for sending it on every turn if desired. To opt into fallback-to-configured behavior, add the- `ReinjectSystemPrompt`capability to your agent.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/ui/vercel-ai
