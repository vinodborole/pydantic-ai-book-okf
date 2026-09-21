---
type: Web Page
title: Storage | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/storage
timestamp: '2026-09-21T12:25:24.826293+00:00'
---

# Storage

“Persistence”, “memory”, “sessions”: several different problems go by those names, and they have different answers. Start here:

| You want to… | Use | Where it lives | 
|---|---|---|
| Save a conversation and pick it up later — a chat thread, a support ticket, an assistant that remembers yesterday | [Serialize the message history](/docs/ai/core-concepts/message-history/#storing-and-loading-messages-to-json) into a column of your own database | Core | 
| Not write the save-and-load code yourself, and get continue-and-fork for free | [`StepPersistence`](https://pydantic.dev/docs/ai/harness/step-persistence/) | [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) | 
| The agent to remember what it learned about someone *across* conversations, not just within one | [`Memory`](https://pydantic.dev/docs/ai/harness/memory/) | [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) | 
| A run to survive the process dying mid-tool-call, and resume exactly where it stopped | [Durable execution](/docs/ai/capabilities/durable_execution/overview/) | Core | 

The first two rows are also the answer to “how do I give my agent memory?” for most of what people mean by it: an agent’s memory of the conversation it is having *is* its message history. There is no separate memory system to add for that — storing the history and passing it back is the whole mechanism. Memory becomes [its own thing](#remembering-across-conversations) only once it has to outlive the thread.

The rows compose: a durable engine keeps one run alive, `StepPersistence` records what each run did, a serialized history is what you hand to the next run, and `Memory` is what’s left when the thread is over. Reaching for a durable engine because you wanted to store a chat thread is the common mistake — a `jsonb` column is enough for that.

Pydantic AI is deliberately unopinionated about your database. It gives you a full-fidelity serialization boundary and leaves the schema to you, because teams’ choices here vary more than the framework can usefully guess: which table the history hangs off, which tenant column it needs, how long you keep it.

The primitive is [`ModelMessagesTypeAdapter`](/docs/ai/core-concepts/message-history/#storing-and-loading-messages-to-json), which round-trips a message history to JSON and back — including fields that are never sent to the model, like a part’s application-only `metadata`. Because that field is typed `Any`, values with no JSON form are normalized on the way through: a `tuple` reloads as a `list`, a `datetime` as its ISO string. The [“What survives a round-trip”](/docs/ai/core-concepts/message-history/#storing-and-loading-messages-to-json) note covers the edges. Store the bytes in a `jsonb` column or equivalent; no schema migration is needed when Pydantic AI adds a message part, because a history serialized by an older version still deserializes.

To store a finished run rather than just its messages — keeping the output, usage, and conversation ID alongside — put an [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) on a Pydantic model of your own and serialize that: see [Storing complete run results](/docs/ai/core-concepts/message-history/#storing-complete-run-results).

[`conversation_id`](/docs/ai/core-concepts/message-history/#correlating-runs-with-run_id-and-conversation_id) is the key to store it under, and appending each run’s [`new_messages()`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.new_messages) rather than rewriting the whole list keeps each write proportional to the turn. [Persisting sessions](/docs/ai/core-concepts/message-history/#persisting-sessions) walks through the pattern.

A frontend on [Vercel AI](/docs/ai/integrations/ui/vercel-ai/) or [AG-UI](/docs/ai/integrations/ui/ag-ui/) keeps a message list of its own, and the adapter that serves it converts in both directions without a request in hand: [`load_messages`](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter.load_messages) turns the protocol’s messages into [`ModelMessage`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelMessage)s, and [`dump_messages`](/docs/ai/api/ui/base/#pydantic_ai.ui.UIAdapter.dump_messages) turns them back.

What the browser sent, as a history an agent can run against — and the shape to store.

What the browser gets back, converted at the edge rather than on the way into the database.

Store the Pydantic AI side and convert at the edge, rather than storing the protocol’s shape. The wire formats have no place for everything a history carries — what each one keeps and drops is spelled out under [Vercel AI message metadata](/docs/ai/integrations/ui/vercel-ai/#message-metadata) and [AG-UI preserving files across round-trips](/docs/ai/integrations/ui/ag-ui/#preserving-files-across-round-trips) — and the fields they drop are the ones the next model request needs.

Messages carry more than they look like they do: [`run_id` and `conversation_id`](/docs/ai/core-concepts/message-history/#correlating-runs-with-run_id-and-conversation_id) are stamped onto each one, so a conversation reloaded from storage stays correlated in [Logfire](/docs/ai/integrations/logfire/) with no bookkeeping of your own, and each run’s span reports that run’s own token usage either way.

What lives outside the messages is [`RunUsage`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage): the conversation’s running total, including [`tool_calls`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage.tool_calls), which no message records. Store it alongside the history and hand it back with `usage=` when [`UsageLimits`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) should budget the whole conversation rather than each run. Carrying it changes nothing about what your traces show: each run’s span reports that run’s own tokens either way, so a conversation’s spend is the sum of its runs.

Nor does a history reach past its own conversation. Replaying yesterday’s threads to give an agent that continuity works until it doesn’t: the prompt grows without bound, every request pays for it, and [compaction](/docs/ai/capabilities/compaction/) drops the parts you were counting on. [Remembering across conversations](#remembering-across-conversations) is a different mechanism.

[`StepPersistence`](https://pydantic.dev/docs/ai/harness/step-persistence/) packages the pattern as a capability you add to an agent, so the load and save calls are not yours to write. It ships in-memory, file, SQLite and MongoDB backends, and its store is a protocol you can implement against your own database.

It records more than the messages: an append-only event log of what the agent did at each boundary, continuable snapshots you can resume or fork a conversation from, and a tool-effect ledger that tells you, after a crash, whether a side effect actually happened.

[`ConversationSearch`](https://pydantic.dev/docs/ai/harness/conversation-search/) builds on that without storing anything of its own: it ranks the history `StepPersistence` already wrote and gives the model a tool to pull earlier turns back into context on demand, including turns [compaction](/docs/ai/capabilities/compaction/) dropped. Pair the two on one store instance and recall needs no extra write path.

Everything above is scoped to a conversation. What an agent knows about someone *between* conversations — their preferences, a decision from last week, a correction they shouldn’t have to repeat — has a different key and a different lifetime.

[`Memory`](https://pydantic.dev/docs/ai/harness/memory/) is the capability for that. It gives the agent a notebook of Markdown files that it writes, reads, and searches through its own tools, and puts a bounded excerpt in each request rather than the whole notebook. It is keyed by a namespace you resolve from your [dependencies](/docs/ai/core-concepts/dependencies/) — usually a user or tenant ID — rather than by `conversation_id`, which is exactly what lets it outlive the thread. Its stores are the durable ones: a file directory, SQLite, PostgreSQL, or one you implement against your own database.

It stays a capability of its own, with a store of its own, rather than something `StepPersistence` writes: a versioned notebook and an append-only run log have little in common. But the two take the same database, so keeping an agent’s notes next to its conversations is one connection and one thing to back up.

Anthropic exposes a memory tool on its own side of the API: [`MemoryTool`](/docs/ai/tools-toolsets/native-tools/#memory-tool) has the model drive a directory of memory files through a tool contract the provider defines, with the storage behind it still yours to supply.

Some providers keep conversation state on their side and reconstruct earlier turns from it, so each request carries only what is new — on the OpenAI Responses API that is [`openai_conversation_id`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModelSettings.openai_conversation_id), covered under [durable conversations](/docs/ai/models/openai/#using-durable-conversations).

Weigh it against a store of your own: it is one provider’s feature, OpenAI documents that earlier input tokens in a chain are still billed, and it is unavailable to organizations with Zero Data Retention enabled.

Pydantic AI does not checkpoint graph execution state, so there is no “rewind to step 4 of a half-finished run and replay from there” inside a single run. Snapshots are taken at settled boundaries between runs, not mid-node. For a run that must survive a crash *while it is executing*, that is what [durable execution](/docs/ai/capabilities/durable_execution/overview/) is for.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/storage
