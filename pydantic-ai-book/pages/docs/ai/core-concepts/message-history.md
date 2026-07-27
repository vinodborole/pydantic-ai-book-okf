---
type: Web Page
title: Messages and chat history | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/message-history
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Messages and chat history

Pydantic AI provides access to messages exchanged during an agent run. These messages can be used both to continue a coherent conversation, and to understand how an agent performed.

After running an agent, you can access the messages exchanged during that run from the `result` object.

Both [ RunResult](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult)
(returned by 

[,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run)

`Agent.run`[) and](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync)

`Agent.run_sync`[(returned by](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult)

`StreamedRunResult`[) have the following methods:](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream)

`Agent.run_stream`- `all_messages()`- `all_messages_json()`
- `new_messages()`- `new_messages_json()`

Example of accessing methods on a [ RunResult](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) :

*(This example is complete, it can be run “as is”)*

Example of accessing methods on a [ StreamedRunResult](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult) :

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The primary use of message histories in Pydantic AI is to maintain context across multiple agent runs.

To use existing messages in a run, pass them to the `message_history` parameter of
[ Agent.run](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run), 

[or](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync)

`Agent.run_sync`[.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream)

`Agent.run_stream`If `message_history` is set and not empty, a new system prompt is not generated — we assume the existing message history includes a system prompt. If your history comes from a source that doesn’t round-trip system prompts (a UI frontend, a database that didn’t persist them, a compaction pipeline), add the [ ReinjectSystemPrompt](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ReinjectSystemPrompt) capability so the agent’s configured 

`system_prompt` is reinjected at the head of the first request when it’s missing.Mid-conversation `SystemPromptPart`s (those in any `ModelRequest` after the first) are sent inline at their original position by providers whose API accepts system messages at arbitrary positions. For providers whose API doesn’t, they’re instead rendered as `<system>`-tagged `UserPromptPart`s at the same position, preserving the prefix cache and positional intent. Leading `SystemPromptPart`s always hoist to the provider’s top-level system parameter.

*(This example is complete, it can be run “as is”)*

Model providers reject a request whose message history has broken tool-call/tool-result pairing — a tool call with no result, or a result with no call. A run that is cancelled or crashes partway through can leave the history in exactly this state, and so can a hand-built, truncated, or context-evicted history. You don’t need to clean these up yourself: before each model request, Pydantic AI repairs the history it was given so the provider accepts it.

The guiding rule is to massage the history into a shape the provider accepts without ever discarding something you meant to send. Repairs only **add** synthesized parts or **remove** parts that are fundamentally unsendable (no provider could accept them); nothing meaningful is silently dropped. Concretely, before each request Pydantic AI:

- **Adds**a synthesized- `ToolReturnPart`- `outcome='interrupted'`- `'failed'`) is not surfaced as a provider error — and carries- `{'pydantic_ai_synthesized_tool_return': True}`in its- `metadata`
- **Removes**an orphaned tool result — a- `ToolReturnPart`- `RetryPromptPart`- `ModelRequest`- `ModelRequest`.

After the invalid parts are handled, consecutive compatible messages are **merged** into one (two adjacent [ ModelRequest](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)s become a single turn, with tool results ordered ahead of user parts). This changes message boundaries but preserves all content, so processed history you inspect afterwards may have fewer messages than you passed in.

The repair is deterministic and idempotent: repairing the same history always produces the same output, running a repaired history through another run leaves it untouched, and synthesized parts contain no wall-clock data, so reuse doesn’t invalidate provider prompt caches.

Tool calls that can still receive a real result are left alone: when the history ends on a `ModelResponse` with tool calls, running without a new `user_prompt` executes them, and [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools) are matched to their `deferred_tool_results` — including when a ‘complete’ `ModelRequest` with the already-executed results follows the response. Repair of that live frontier only happens when the interruption is evident: a final response with [ state='interrupted'](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.state) or a trailing request with 

[(e.g. from a](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest.state)

`state='interrupted'`[cancelled stream](/docs/ai/core-concepts/output#cancelling-streams)or a crash during tool execution) whose tool calls will never be executed.

This pipeline handles regular, locally-executed tool calls only. Builtin (server-side) tool parts — produced and resulted by the provider inline — are left untouched and repaired by each model’s own serializer instead. Some other provider-invalid shapes are also out of scope and may be rejected: duplicate tool results for one call, and provider-specific ordering rules beyond call/result pairing.

Each `ModelRequest` and `ModelResponse` carries two identifiers:

- `run_id`- `RunContext.run_id`- `AgentRunResult.run_id`- `gen_ai.agent.call.id`.
- `conversation_id`- `message_history`. Also available as- `AgentRunResult.conversation_id`- `gen_ai.conversation.id`.

A fresh `run_id` is generated for every agent run (or you can pass `run_id='<your-id>'` to use an ID minted by your application — e.g. one created, stored, or handed out to a client before the run starts). Unlike `conversation_id`, `run_id` is **never** inherited from `message_history`. Each [ Agent.run](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) call — including a 

[deferred-tool resume](/docs/ai/tools-toolsets/deferred-tools)— is a separate run with its own

`run_id`. Passing an empty `run_id=''`, or a `run_id` that already appears on `message_history`, raises [, because both break](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError)

`UserError`[boundary detection. Correlate pause/resume or multi-turn work with](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.new_messages)

`new_messages()``conversation_id` instead. When retrying a failed run with the same `run_id`, rebuild `message_history` without the failed attempt’s messages.A fresh `conversation_id` is generated on the first run, stamped onto every message produced by that run, and inherited by subsequent runs that pass the messages back via `message_history`. This means you can correlate traces from a multi-turn conversation in [Logfire](/docs/ai/integrations/logfire) (or any OpenTelemetry backend) without tracking anything yourself — as long as the message history round-trips, the conversation ID does too.

To override or fork `conversation_id`:

- Pass `conversation_id='<your-id>'`to use an ID from your own application (e.g. a chat thread ID stored in your database).
- Pass `conversation_id='new'`to start a fresh conversation that ignores any`conversation_id`already on`message_history`— useful for branching off an existing thread without making the caller generate an ID.

The [UI adapters](/docs/ai/integrations/ui/overview) auto-populate `conversation_id` from the protocol’s own thread/chat ID, so frontends using these protocols get conversation correlation for free. Protocol-level run IDs (for example AG-UI’s `runId`) are **not** mapped into the agent’s `run_id` — pass `run_id=` explicitly on `AGUIAdapter.run_stream` / `dispatch_request` (or a plain `Agent.run`) if you need them to match.

While maintaining conversation state in memory is enough for many applications, often times you may want to store the messages history of an agent run on disk or in a database. This might be for evals, for sharing data between Python and JavaScript/TypeScript, or any number of other use cases.

The intended way to do this is using a `TypeAdapter`.

We export [ ModelMessagesTypeAdapter](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelMessagesTypeAdapter) that can be used for this, or you can create your own.

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

[ sanitize_messages](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.sanitize_messages) applies the same default message sanitization used by the 

[UI adapters](/docs/ai/integrations/ui/overview): it strips client-supplied system prompts, drops non-HTTP file URL schemes, resets non-allowlisted

[values to](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FileUrl.force_download)

`FileUrl.force_download``False`, drops uploaded file references, and removes unresolved tool calls at the end of the history.Each sanitization can be turned off individually when the corresponding parts were created by trusted server-side code: pass `strip_system_prompts=False`, add schemes to `allowed_file_url_schemes`, add values to `allowed_file_url_force_download`, or set `allow_uploaded_files=True`. See [file URL input security](/docs/ai/core-concepts/input#user-side-download-vs-direct-file-url) for the file input trust model.

Since messages are defined by simple dataclasses, you can manually create and manipulate, e.g. for testing.

The message format is independent of the model used, so you can use messages in different agents, or the same agent with different models.

In the example below, we reuse the message from the first agent run, which uses the `openai:gpt-5.2` model, in a second agent run using the `google:gemini-3-pro-preview` model.

*(This example is complete, it can be run “as is”)*

The same `message_history` parameter also works when the next run uses a
different [ Agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent). This is useful for

[programmatic agent hand-off](/docs/ai/guides/multi-agent-applications#programmatic-agent-hand-off), where your application runs one agent, then gives another agent the conversation so far as context.

*(This example is complete, it can be run “as is”)*

For more complex multi-agent patterns, see the [multi-agent applications](/docs/ai/guides/multi-agent-applications) documentation.

To change the conversation mid-run, build *new* message objects rather than modifying existing ones: [inject new messages](#injecting-messages-mid-run) with `enqueue`, or prune, summarize, or otherwise rewrite the history the model receives with a [history processor](#processing-message-history). When you need to edit an earlier message — say, compacting a large tool output — copy it with [ dataclasses.replace](https://docs.python.org/3/library/dataclasses.html#dataclasses.replace), passing a new 

`parts` list of new (or reused) part objects; edited parts are likewise built with `replace` rather than modified. Replacing a message in the history and reassigning its `parts` list are both safe.Tools, capability hooks, and external code driving an agent run can inject extra content
into the conversation mid-run with [ RunContext.enqueue](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.enqueue)
(when a 

`RunContext` is in scope, e.g. inside a tool or capability hook) or
[(from external code driving](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.enqueue)

`AgentRun.enqueue`[). Use this when something happens during a run that the agent should know about — a tool wants to add follow-up context, an external event needs to](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.iter)

`agent.iter()`*steer*the agent’s plan, or background work needs to reach the agent when it completes.

A `priority` controls when the enqueued content is delivered:

- `'asap'`(default): delivered at the earliest opportunity — added to the next- `ModelRequest`- **steering**an in-flight agent.
- `'when_idle'`: delivered only when the agent would otherwise terminate, after any- `'asap'`messages. Use when the agent shouldn’t be interrupted but should pick up the new work — a follow-up task — once it’s done with what it’s doing.

`enqueue` is variadic — each positional argument is one item, and can be:

- a piece of `UserContent``str`or multi-modal content like an`ImageUrl``UserPromptPart``enqueue('caption', image)`forms one user turn. To pass an existing list, spread it:`enqueue(*items)`;
- a `ModelRequestPart``SystemPromptPart`
- a complete `ModelRequest``ModelResponse``instructions`/`metadata`or to inject a synthetic prior turn.

Adjacent part-style items (user content and [ ModelRequestPart](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequestPart)s) are coalesced into one 

[; complete messages stay separate. This lets a single call inject an interleaved exchange — for example a synthetic tool call (a](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)

`ModelRequest`[) followed by its result (a](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse)

`ModelResponse`[). The content must end in a request, so the agent has something to respond to.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)

`ModelRequest`Both `enqueue` methods return an `enqueue_id` (`str`) for a non-empty call, or `None` when called with no content. When the queued content is actually delivered into run history, the [event stream](/docs/ai/core-concepts/agent#streaming-all-events) yields an [ EnqueuedMessagesEvent](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.EnqueuedMessagesEvent) carrying that 

`enqueue_id` and the delivered messages (exactly as they landed in history), so a client can observe when its steering message took effect. The event carries the delivered message objects themselves — the same objects held in the run’s message history. A history processor that replaces history with new message objects does not affect the event, but [in-place mutation](#editing-existing-messages)of a delivered message will be visible through it.

Use [ RunContext.enqueue](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.enqueue) when you have a

`RunContext` in scope:The `'asap'` message is appended to the agent’s message history and is visible to the
model on the next request, alongside any tool returns from the same step. A
[ SystemPromptPart](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.SystemPromptPart) is delivered the same way; on
providers that hoist system prompts (e.g. Anthropic, Google) a non-leading one is sent as a

`<system>`-tagged user-role message, so it keeps its mid-conversation position rather than being
lifted to the top.Use [ AgentRun.enqueue](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.enqueue) when you’re driving a run
from outside (e.g. forwarding events from a webhook, chat platform, or job queue):

The example drives the run with [ agent.iter()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.iter) +

[because](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next)

`AgentRun.next()``'when_idle'` messages are only
drained when the agent would otherwise reach an `End` — that drain happens in `after_node_run`,
which doesn’t fire inside a bare `async for node in agent_run:` loop. `'asap'` messages are
drained in `before_model_request` (which fires either way) and also at the same end-of-run point
if anything arrived during the final step. Reaching the end of a bare `async for` loop with
undrained pending messages raises [, since those messages would otherwise be silently lost.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UndrainedPendingMessagesError)

`UndrainedPendingMessagesError`Sometimes you may want to modify the message history before it’s sent to the model. This could be for privacy reasons (filtering out sensitive information), to save costs on tokens, to give less context to the LLM, or custom processing logic.

Pydantic AI provides the [ ProcessHistory](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessHistory) capability that allows
you to intercept and modify the message history before each model request.

Each [ ProcessHistory](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessHistory) wraps a callable that takes a list of

[and returns a modified list of the same type.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelMessage)

`ModelMessage`Each processor is applied in sequence, and processors can be either synchronous or asynchronous.

You can use the `history_processor` to only keep the recent messages:

History processors can optionally accept a [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) parameter to access
additional information about the current run, such as dependencies, model information, and usage statistics:

This allows for more sophisticated message processing based on the current state of the agent run.

Use an LLM to summarize older messages to preserve context while reducing tokens. This is one of several ways to keep a conversation within the context window — see [Compaction](/docs/ai/capabilities/compaction) for the full picture, including provider-native compaction and ready-made strategies from [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/compaction/).

You can test what messages are actually sent to the model provider using
[ FunctionModel](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel):

You can also use multiple processors:

In this case, the `filter_responses` processor will be applied first, and the
`summarize_old_messages` processor will be applied second.

For a more complete example of using messages in conversations, see the [chat app](/docs/ai/examples/conversational-agents/chat-app) example.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/message-history
