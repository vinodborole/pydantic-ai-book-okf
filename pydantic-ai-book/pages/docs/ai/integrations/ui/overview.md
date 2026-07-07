---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/ui/overview
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Overview

If you’re building a chat app or other interactive frontend for an AI agent, your backend will need to receive agent run input (like a chat message or complete message history) from the frontend, and will need to stream the agent’s events (like text, thinking, and tool calls) to the frontend so that the user knows what’s happening in real time.

While your frontend could use Pydantic AI’s `ModelRequest` and `AgentStreamEvent` directly, you’ll typically want to use a UI event stream protocol that’s natively supported by your frontend framework.

Pydantic AI natively supports two UI event stream protocols:

These integrations are implemented as subclasses of the abstract `UIAdapter` class, so they also serve as a reference for integrating with other UI event stream protocols.

The protocol-specific `UIAdapter` subclass (i.e. `AGUIAdapter` or `VercelAIAdapter`) is responsible for transforming agent run input received from the frontend into arguments for `Agent.run_stream_events()`, running the agent, and then transforming Pydantic AI events into protocol-specific events. The event stream transformation is handled by a protocol-specific `UIEventStream` subclass, but you typically won’t use this directly.

If you’re using a Starlette-based web framework like FastAPI, you can use the `UIAdapter.dispatch_request()` class method from an endpoint function to directly handle a request and return a streaming response of protocol-specific events. This is demonstrated in the next section.

If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `UIAdapter` instance and directly use its methods. This is demonstrated in “Advanced Usage” section below.

Besides the request, `UIAdapter.dispatch_request()` takes the agent, the same optional arguments as `Agent.run_stream_events()`, and an optional `on_complete` callback function that receives the completed `AgentRunResult` and can optionally yield additional protocol-specific events.

If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `UIAdapter` instance and directly use its methods, which can be chained to accomplish the same thing as the `UIAdapter.dispatch_request()` class method shown above:

- The `UIAdapter.build_run_input()`class method takes the request body as bytes and returns a protocol-specific run input object, which you can then pass to the`UIAdapter()`constructor along with the agent.- You can also use the `UIAdapter.from_request()`class method to build an adapter directly from a Starlette/FastAPI request.
 
- You can also use the 
- The `UIAdapter.run_stream()`method runs the agent and returns a stream of protocol-specific events. It supports the same optional arguments as`Agent.run_stream_events()`and an optional`on_complete`callback function that receives the completed`AgentRunResult`and can optionally yield additional protocol-specific events.- You can also use `UIAdapter.run_stream_native()`to run the agent and return a stream of Pydantic AI events instead, which can then be transformed into protocol-specific events using`UIAdapter.transform_stream()`.
 
- You can also use 
- The `UIAdapter.encode_stream()`method encodes the stream of protocol-specific events as SSE (HTTP Server-Sent Events) strings, which you can then return as a streaming response.- You can also use `UIAdapter.streaming_response()`to generate a Starlette/FastAPI streaming response directly from the protocol-specific event stream returned by`run_stream()`.
 
- You can also use 

UI adapter endpoints aren’t authentication boundaries. Both the AG-UI and Vercel AI protocols are designed around the client transmitting the full conversation history on each request, so anything in `message_history` from the protocol — assistant messages, tool calls, file URLs, tool results — is under the caller’s control. Treat the adapter endpoint as an internal backend service, running it inside your own authenticated route handler. See the AG-UI security considerations page for more on the deployment model both protocols assume.

The adapters apply a few defaults so that the authoritative state stays on your side:

- **System prompts**— client-submitted- `SystemPromptPart`s are stripped by default and replaced with the agent’s configured prompt. Control with- `UIAdapter.manage_system_prompt`; see each adapter’s docs for details.
- **Dangling tool calls**— if the client-submitted history ends in a- `ModelResponse`with unresolved- `ToolCallPart`s and no matching- `deferred_tool_results`, the tool calls are dropped with a warning so the agent doesn’t execute tool calls that the model never emitted. For human-in-the-loop resumption, pass explicit- `deferred_tool_results`to the run method — tool calls resolved by those results are kept.
- **File URL schemes**— only- `http`and- `https`are accepted by default for- `FileUrl`parts in client-submitted messages. Non-HTTP schemes like- `s3://`or- `gs://`are dropped, since they cause the provider to fetch the object using your server’s IAM role or service account. See- `UIAdapter.allowed_file_url_schemes`.
- **File URL download mode**—- `FileUrl.force_download`values other than- `False`are reset to- `False`by default on client-submitted messages. This prevents clients from forcing the server to fetch a URL, or using- `'allow-local'`to opt out of the SSRF private-IP block. After auditing your frontend, opt into additional values with- `UIAdapter.allowed_file_url_force_download`.
- **Uploaded files**— client-submitted- `UploadedFile`parts are dropped by default, just like non-HTTP- `FileUrl`s, since the server resolves them against the provider’s file storage API using its own credentials. After auditing your frontend, honor them by setting- `UIAdapter.allow_uploaded_files`to- `True`. This is a purely inbound security setting: file content the agent produces is always serialized on the way back out to the client.

For stricter conversation integrity (e.g. ensuring prior assistant turns and tool returns match what the server actually produced), persist the history server-side keyed by the thread/session ID and pass it to the adapter via `message_history` — caller-supplied history is trusted as coming from server-side persistence and isn’t subject to this sanitization.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/ui/overview
