---
type: Web Page
title: Messages and chat history | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/message-history
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Messages and chat history

Pydantic AI provides access to messages exchanged during an agent run. These messages can be used both to continue a coherent conversation, and to understand how an agent performed.

After running an agent, you can access the messages exchanged during that run from the `result` object.

Both `RunResult`
(returned by `Agent.run`, `Agent.run_sync`)
and `StreamedRunResult` (returned by `Agent.run_stream`) have the following methods:

- `all_messages()`: returns all messages, including messages from prior runs. There’s also a variant that returns JSON bytes,- `all_messages_json()`.
- `new_messages()`: returns only the messages from the current run. There’s also a variant that returns JSON bytes,- `new_messages_json()`.

Example of accessing methods on a `RunResult` :

*(This example is complete, it can be run “as is”)*

Example of accessing methods on a `StreamedRunResult` :

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The primary use of message histories in Pydantic AI is to maintain context across multiple agent runs.

To use existing messages in a run, pass them to the `message_history` parameter of
`Agent.run`, `Agent.run_sync` or
`Agent.run_stream`.

If `message_history` is set and not empty, a new system prompt is not generated — we assume the existing message history includes a system prompt. If your history comes from a source that doesn’t round-trip system prompts (a UI frontend, a database that didn’t persist them, a compaction pipeline), add the `ReinjectSystemPrompt` capability so the agent’s configured `system_prompt` is reinjected at the head of the first request when it’s missing.

Mid-conversation `SystemPromptPart`s (those in any `ModelRequest` after the first) are sent inline at their original position by providers whose API accepts system messages at arbitrary positions. For providers whose API doesn’t, they’re instead rendered as `<system>`-tagged `UserPromptPart`s at the same position, preserving the prefix cache and positional intent. Leading `SystemPromptPart`s always hoist to the provider’s top-level system parameter.

*(This example is complete, it can be run “as is”)*

Each `ModelRequest` and `ModelResponse` carries two identifiers:

- `run_id`— unique per agent run; emitted on the OpenTelemetry agent run span as- `gen_ai.agent.call.id`.
- `conversation_id`— shared across all runs that build on the same- `message_history`; emitted as- `gen_ai.conversation.id`.

A fresh `conversation_id` is generated on the first run, stamped onto every message produced by that run, and inherited by subsequent runs that pass the messages back via `message_history`. This means you can correlate traces from a multi-turn conversation in Logfire (or any OpenTelemetry backend) without tracking anything yourself — as long as the message history round-trips, the conversation ID does too.

To override or fork:

- Pass `conversation_id='<your-id>'`to use an ID from your own application (e.g. a chat thread ID stored in your database).
- Pass `conversation_id='new'`to start a fresh conversation that ignores any`conversation_id`already on`message_history`— useful for branching off an existing thread without making the caller generate an ID.

The UI adapters auto-populate `conversation_id` from the protocol’s own thread/chat ID, so frontends using these protocols get correlation for free.

While maintaining conversation state in memory is enough for many applications, often times you may want to store the messages history of an agent run on disk or in a database. This might be for evals, for sharing data between Python and JavaScript/TypeScript, or any number of other use cases.

The intended way to do this is using a `TypeAdapter`.

We export `ModelMessagesTypeAdapter` that can be used for this, or you can create your own.

Here’s an example showing how:

Alternatively, you can create a `TypeAdapter` from scratch:

```
from pydantic import TypeAdapter
from pydantic_ai import ModelMessage
ModelMessagesTypeAdapter = TypeAdapter(list[ModelMessage])
```
Alternatively you can serialize to/from JSON directly:

```
from pydantic_core import to_json
...
as_json_objects = to_json(history_step_1)
same_history_as_step_1 = ModelMessagesTypeAdapter.validate_json(as_json_objects)
```
You can now continue the conversation with history `same_history_as_step_1` despite creating a new agent run.

*(This example is complete, it can be run “as is”)*

The `message_history` parameter is trusted server-side state. If you load history that came from a browser request or another untrusted boundary, sanitize it before passing it to the agent.

`sanitize_messages` applies the same default message sanitization used by the UI adapters: it strips client-supplied system prompts, drops non-HTTP file URL schemes, resets non-allowlisted `FileUrl.force_download` values to `False`, drops uploaded file references, and removes unresolved tool calls at the end of the history.

Each sanitization can be turned off individually when the corresponding parts were created by trusted server-side code: pass `strip_system_prompts=False`, add schemes to `allowed_file_url_schemes`, add values to `allowed_file_url_force_download`, or set `allow_uploaded_files=True`. See file URL input security for the file input trust model.

Since messages are defined by simple dataclasses, you can manually create and manipulate, e.g. for testing.

The message format is independent of the model used, so you can use messages in different agents, or the same agent with different models.

In the example below, we reuse the message from the first agent run, which uses the `openai:gpt-5.2` model, in a second agent run using the `google:gemini-3-pro-preview` model.

*(This example is complete, it can be run “as is”)*

The same `message_history` parameter also works when the next run uses a
different `Agent`. This is useful for
programmatic agent hand-off,
where your application runs one agent, then gives another agent the conversation
so far as context.

*(This example is complete, it can be run “as is”)*

For more complex multi-agent patterns, see the multi-agent applications documentation.

Tools, capability hooks, and external code driving an agent run can inject extra content
into the conversation mid-run with `RunContext.enqueue`
(when a `RunContext` is in scope, e.g. inside a tool or capability hook) or
`AgentRun.enqueue` (from external code driving
`agent.iter()`). Use this when something happens during a
run that the agent should know about — a tool wants to add follow-up context, an external event
needs to *steer* the agent’s plan, or background work needs to reach the agent when it completes.

A `priority` controls when the enqueued content is delivered:

- `'asap'`(default): delivered at the earliest opportunity — added to the next- `ModelRequest`, or, if the agent would otherwise terminate before another request, used to redirect the run into one more request. Use when the new context should reach the model as soon as possible; this is what other frameworks often call- **steering**an in-flight agent.
- `'when_idle'`: delivered only when the agent would otherwise terminate, after any- `'asap'`messages. Use when the agent shouldn’t be interrupted but should pick up the new work — a follow-up task — once it’s done with what it’s doing.

`enqueue` is variadic — each positional argument is one item, and can be:

- a piece of `UserContent`— a`str`or multi-modal content like an`ImageUrl`. Adjacent user content is gathered into a single`UserPromptPart`, so`enqueue('caption', image)`forms one user turn. To pass an existing list, spread it:`enqueue(*items)`;
- a `ModelRequestPart`, such as a`SystemPromptPart`;
- a complete `ModelRequest`or`ModelResponse`, to control request-level fields like`instructions`/`metadata`or to inject a synthetic prior turn.

Adjacent part-style items (user content and `ModelRequestPart`s) are coalesced into one `ModelRequest`; complete messages stay separate. This lets a single call inject an interleaved exchange — for example a synthetic tool call (a `ModelResponse`) followed by its result (a `ModelRequest`). The content must end in a request, so the agent has something to respond to.

Use `RunContext.enqueue` when you have a
`RunContext` in scope:

The `'asap'` message is appended to the agent’s message history and is visible to the
model on the next request, alongside any tool returns from the same step. A
`SystemPromptPart` is delivered the same way; on
providers that hoist system prompts (e.g. Anthropic, Google) a non-leading one is sent as a
`<system>`-tagged user-role message, so it keeps its mid-conversation position rather than being
lifted to the top.

Use `AgentRun.enqueue` when you’re driving a run
from outside (e.g. forwarding events from a webhook, chat platform, or job queue):

The example drives the run with `agent.iter()` +
`AgentRun.next()` because `'when_idle'` messages are only
drained when the agent would otherwise reach an `End` — that drain happens in `after_node_run`,
which doesn’t fire inside a bare `async for node in agent_run:` loop. `'asap'` messages are
drained in `before_model_request` (which fires either way) and also at the same end-of-run point
if anything arrived during the final step. Reaching the end of a bare `async for` loop with
undrained pending messages raises `UndrainedPendingMessagesError`,
since those messages would otherwise be silently lost.

Sometimes you may want to modify the message history before it’s sent to the model. This could be for privacy reasons (filtering out sensitive information), to save costs on tokens, to give less context to the LLM, or custom processing logic.

Pydantic AI provides the `ProcessHistory` capability that allows
you to intercept and modify the message history before each model request.

Each `ProcessHistory` wraps a callable that takes a list of
`ModelMessage` and returns a modified list of the same type.

Each processor is applied in sequence, and processors can be either synchronous or asynchronous.

You can use the `history_processor` to only keep the recent messages:

History processors can optionally accept a `RunContext` parameter to access
additional information about the current run, such as dependencies, model information, and usage statistics:

This allows for more sophisticated message processing based on the current state of the agent run.

Use an LLM to summarize older messages to preserve context while reducing tokens.

You can test what messages are actually sent to the model provider using
`FunctionModel`:

You can also use multiple processors:

In this case, the `filter_responses` processor will be applied first, and the
`summarize_old_messages` processor will be applied second.

For a more complete example of using messages in conversations, see the chat app example.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/message-history
