---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/ui/overview
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Overview

If you’re building a chat app or other interactive frontend for an AI agent, your backend will need to receive agent run input (like a chat message or complete [message history](/docs/ai/core-concepts/message-history)) from the frontend, and will need to stream the [agent’s events](/docs/ai/core-concepts/agent#streaming-all-events) (like text, thinking, and tool calls) to the frontend so that the user knows what’s happening in real time.

While your frontend could use Pydantic AI’s [ ModelRequest](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest) and 

[directly, you’ll typically want to use a UI event stream protocol that’s natively supported by your frontend framework.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)

`AgentStreamEvent`Pydantic AI natively supports two UI event stream protocols:

These integrations are implemented as subclasses of the abstract [ UIAdapter](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter) class, so they also serve as a reference for integrating with other UI event stream protocols.

The protocol-specific [ UIAdapter](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter) subclass (i.e. 

[or](/docs/ai/api/ui/ag_ui/#pydantic_ai.ui.ag_ui.AGUIAdapter)

`AGUIAdapter`[) is responsible for transforming agent run input received from the frontend into arguments for](/docs/ai/api/ui/vercel_ai/#pydantic_ai.ui.vercel_ai.VercelAIAdapter)

`VercelAIAdapter`[, running the agent, and then transforming Pydantic AI events into protocol-specific events. The event stream transformation is handled by a protocol-specific](/docs/ai/core-concepts/agent#running-agents)

`Agent.run_stream_events()`[subclass, but you typically won’t use this directly.](/docs/ai/api/ui/base/#pydantic_ai.ui.UIEventStream)

`UIEventStream`If you’re using a Starlette-based web framework like FastAPI, you can use the [ UIAdapter.dispatch_request()](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter.dispatch_request) class method from an endpoint function to directly handle a request and return a streaming response of protocol-specific events. This is demonstrated in the next section.

If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `UIAdapter` instance and directly use its methods. This is demonstrated in “Advanced Usage” section below.

Besides the request, [ UIAdapter.dispatch_request()](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter.dispatch_request) takes the agent, the same optional arguments as 

[, and an optional](/docs/ai/core-concepts/agent#running-agents)

`Agent.run_stream_events()``on_complete` callback function that receives the completed [and can optionally yield additional protocol-specific events.](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult)

`AgentRunResult`If you’re using a web framework not based on Starlette (e.g. Django or Flask) or need fine-grained control over the input or output, you can create a `UIAdapter` instance and directly use its methods, which can be chained to accomplish the same thing as the `UIAdapter.dispatch_request()` class method shown above:

- The `UIAdapter.build_run_input()``UIAdapter()`- You can also use the `UIAdapter.from_request()`
 
- You can also use the 
- The `UIAdapter.run_stream()``Agent.run_stream_events()``on_complete`callback function that receives the completed`AgentRunResult`- You can also use `UIAdapter.run_stream_native()``UIAdapter.transform_stream()`
 
- You can also use 
- The `UIAdapter.encode_stream()`- You can also use `UIAdapter.streaming_response()``run_stream()`.
 
- You can also use 

UI adapter endpoints aren’t authentication boundaries. Both the AG-UI and Vercel AI protocols are designed around the client transmitting the full conversation history on each request, so anything in `message_history` from the protocol — assistant messages, tool calls, file URLs, tool results — is under the caller’s control. Treat the adapter endpoint as an internal backend service, running it inside your own authenticated route handler. See the [AG-UI security considerations](https://learn.microsoft.com/en-us/agent-framework/integrations/ag-ui/security-considerations) page for more on the deployment model both protocols assume.

The adapters apply a few defaults so that the authoritative state stays on your side:

- **System prompts**— client-submitted- `SystemPromptPart`- `UIAdapter.manage_system_prompt`
- **Dangling tool calls**— if the client-submitted history ends in a- `ModelResponse`- `ToolCallPart`- `deferred_tool_results`, the tool calls are dropped with a warning so the agent doesn’t execute tool calls that the model never emitted. For human-in-the-loop resumption, pass explicit- `deferred_tool_results`to the run method — tool calls resolved by those results are kept.
- **File URL schemes**— only- `http`and- `https`are accepted by default for- `FileUrl`- `s3://`or- `gs://`are dropped, since they cause the provider to fetch the object using your server’s IAM role or service account. See- `UIAdapter.allowed_file_url_schemes`
- **File URL download mode**—- `FileUrl.force_download`- `False`are reset to- `False`by default on client-submitted messages. This prevents clients from forcing the server to fetch a URL, or using- `'allow-local'`to opt out of the SSRF private-IP block. After auditing your frontend, opt into additional values with- `UIAdapter.allowed_file_url_force_download`
- **Uploaded files**— client-submitted- `UploadedFile`- `FileUrl`s, since the server resolves them against the provider’s file storage API using its own credentials. After auditing your frontend, honor them by setting- `UIAdapter.allow_uploaded_files`- `True`. This is a purely inbound security setting: file content the agent produces is always serialized on the way back out to the client.

For stricter conversation integrity (e.g. ensuring prior assistant turns and tool returns match what the server actually produced), persist the history server-side keyed by the thread/session ID and pass it to the adapter via `message_history` — caller-supplied history is trusted as coming from server-side persistence and isn’t subject to this sanitization.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/ui/overview
