---
type: Web Page
title: Conversation Search | Pydantic Docs
description: BM25-search the history StepPersistence already stores -- turns that
  compaction dropped from the live context, and past runs in the same store.
resource: https://pydantic.dev/docs/ai/harness/conversation-search
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Conversation Search

`ConversationSearch` gives the model a `search_conversation_history` tool that BM25-ranks the history a `StepPersistence` capability already persists — earlier turns that compaction dropped from the live context, and past runs in the same store.

The API may change between releases. Where practical, breaking changes ship with a deprecation warning.

Compaction capabilities (`SlidingWindowCompaction`, `SummarizingCompaction`, …) narrow the live history so it fits the context window. `SummarizingCompaction` persists its edits: once a prefix is replaced by a summary, the originals are gone from the run’s `message_history` on the next turn. The model can no longer recall an exact file path, a decision, or a value stated earlier — only the summary’s paraphrase of it. And nothing at all from previous runs is reachable, however well persisted.

`ConversationSearch` persists nothing itself. It reads whatever a persistence capability already stores, through a `HistorySource`, and exposes one tool, `search_conversation_history`, that BM25-ranks that history so the model can pull exact details back into context on demand.

The shipped source, `SnapshotHistorySource`, reads the snapshots `StepPersistence` writes: pair the two capabilities on a shared store instance and recall works with no extra write path, no ordering constraints, and no hook coordination.

```
from pydantic_ai import Agent
from pydantic_ai_harness.compaction import SlidingWindowCompaction
from pydantic_ai_harness.conversation_search import ConversationSearch, SnapshotHistorySource
from pydantic_ai_harness.step_persistence import SqliteStepStore, StepPersistence
store = SqliteStepStore(database='sessions.db')
agent = Agent(
    'openai:gpt-5',
    capabilities=[
        StepPersistence(store=store),
        ConversationSearch(SnapshotHistorySource(store)),
        SlidingWindowCompaction(max_messages=40),
    ],
)
```
- Ranking is BM25 (the algorithm behind Lucene/Elasticsearch), implemented in pure Python — no new dependencies. Rare terms and exact matches score higher; multi-word queries score each word independently.
- Results carry provenance (`run: ... | conversation: ...` ), and the tool’s optional`run_id` argument scopes a search to one run — so a run referenced elsewhere (for example by a compaction receipt’s transcript handle) is directly resolvable.
- The search reads the store lazily at call time, so it always sees everything persisted so far, including earlier steps of the current run.

`StepPersistence` saves a full-history snapshot at every step boundary. A compaction strategy that persists its edits (like `SummarizingCompaction`) carries those edits into *later* snapshots — but the earlier snapshots of the same run were taken while the originals were still live. `SnapshotHistorySource` unions each run’s snapshots in write order, skips derived summary artifacts, and removes the overlap between the accumulated history’s suffix and each snapshot’s prefix. This recovers the originals plus everything compaction never touched while preserving repeated messages at distinct sequence positions — as far back as the store still retains those pre-compaction snapshots (see Limitations). Only `complete` snapshots contribute: `interrupted` captures (unsettled tool work, synthesized tool returns) are excluded by the stores’ default read gate.

Overlap matching keys off a content hash of each serialized message, not object identity: consecutive snapshots re-serialize the same growing history, and durable executors (Temporal, DBOS) re-instantiate messages between steps.

`HistorySource` is deliberately substrate-neutral (“enumerate runs, yield each run’s durable message record”): a persistence substrate that keeps an append-only entry log can implement it directly by replay, replacing the snapshot-union adapter without touching the search layer.

A store is a single corpus. With the default `scope='all'`, one `search_conversation_history` call ranks every run the source enumerates and can return verbatim excerpts from any of them — which is the cross-session recall [#124](https://github.com/pydantic/pydantic-ai-harness/issues/124) asks for, and the right default when the store holds one principal’s history.

If several users or tenants share a store, set `scope='conversation'`:

```
agent = Agent(
    'openai:gpt-5',
    capabilities=[
        StepPersistence(store=store),
        ConversationSearch(SnapshotHistorySource(store), scope='conversation'),
    ],
)
async def ask(question: str, user_id: str) -> str:
    result = await agent.run(question, conversation_id=user_id)
    return result.output
```
The corpus is then restricted to runs whose `conversation_id` matches the calling run’s, the tool’s own description tells the model the restriction applies, and the tool’s `run_id` argument cannot reach past it — an out-of-scope run reports the same “no persisted history” answer as a run that does not exist.

A run with no `conversation_id` searches nothing under this scope and the tool says why. Matching on “conversation id is unset” would pool every unlabelled run in the store into one corpus, which is the exposure the scope exists to prevent, so it fails closed instead. Pass `conversation_id=` to `Agent.run(...)` (the same value `StepPersistence` records on the run).

Scoping is applied to the `RunRecord`s a `HistorySource` returns, so a custom source must populate `conversation_id` on them for `scope='conversation'` to match anything.

| Option | Default | Purpose | 
|---|---|---|
| `source` | (required) | Where the corpus comes from. Use `SnapshotHistorySource(store)` over the store`StepPersistence` writes to. | 
| `scope` | `'all'` | How much of the store one search may reach: `'all'` , or`'conversation'` to restrict it to the calling run’s`conversation_id` . See Scope. | 
| `max_matches` | `10` | Maximum matching excerpts the search tool returns. | 
| `context_lines` | `5` | Lines shown around each match (within the match’s run). | 
| `bm25_k1` | `1.5` | BM25 term-frequency saturation. This capability’s default; Lucene’s `BM25Similarity` uses`1.2` . | 
| `bm25_b` | `0.75` | BM25 length normalization (Lucene default). | 
| `add_instructions` | `True` | Emit a short note telling the model the recall tool exists. | 
| `tool_id` | `conversation-search` | Toolset id for the search tool. | 

- Search only reaches what was persisted: history inherited from runs that never ran with `StepPersistence` (for example a long`message_history` passed in from an unpersisted session) cannot be recovered if compaction drops it before the first snapshot.
- Recovery of compaction-dropped originals depends on the pre-compaction snapshots still being retained. A store with bounded snapshot retention (for example a per-run snapshot cap) can prune the early snapshots that held those originals; a search then returns only what the surviving snapshots still carry, degrading to a partial result rather than erroring. Retain full snapshot history for the run if complete recovery matters.
- The corpus is rebuilt on each tool call by reading every run’s snapshots. Snapshot storage is cumulative (each snapshot re-serializes the growing history), so large stores make each search proportionally more expensive. A persistent index (SQLite FTS5, tracked in [#124](https://github.com/pydantic/pydantic-ai-harness/issues/124) ) is the scaling path.
- Reading snapshots restores externalized media (large binary payloads) even though the text index never uses it; stores with remote media backends pay that fetch cost per search.

- [Pydantic AI capabilities](/docs/ai/capabilities/overview/)
- [Step Persistence](/docs/ai/harness/step-persistence) — the substrate this capability reads
- [Compaction](/docs/ai/harness/compaction) — the capabilities whose drops this one recovers from

**Bases:** `AbstractCapability[AgentDepsT]`

Search persisted conversation history with a dependency-free BM25 tool.

This capability persists nothing itself: it reads whatever history a persistence
capability already stores, through a `HistorySource`. Pair it with
`StepPersistence` sharing the same store, and the model can recall what
compaction dropped from the live context as well as anything from past runs:

```
from pydantic_ai import Agent
from pydantic_ai_harness.compaction import SlidingWindowCompaction
from pydantic_ai_harness.conversation_search import ConversationSearch, SnapshotHistorySource
from pydantic_ai_harness.step_persistence import SqliteStepStore, StepPersistence
store = SqliteStepStore(database='sessions.db')
agent = Agent(
    'openai:gpt-5',
    capabilities=[
        StepPersistence(store=store),
        ConversationSearch(SnapshotHistorySource(store)),
        SlidingWindowCompaction(max_messages=40),
    ],
)
```
Some compaction strategies persist their edits into the run’s durable message
history (`SummarizingCompaction` replaces summarized prefixes for good; a
`SlidingWindowCompaction` trim only narrows what each request sends). Either way,
`StepPersistence` snapshots each step boundary before the next compaction runs,
so the union of a run’s snapshots still holds the originals —
`SnapshotHistorySource` recovers them. No ordering or hook coordination between
the capabilities is required; the search tool reads the store lazily at call
time.

Where the search corpus comes from. Use `SnapshotHistorySource` over the
store a `StepPersistence` capability writes to.

**Type:** `HistorySource`

How much of the store one search may reach.

`all` searches every run the source enumerates. `conversation` restricts the
corpus to runs whose `conversation_id` matches the calling run’s, which a store
shared across users or tenants needs: with `all`, any run reading that store can
retrieve verbatim excerpts from every other conversation in it. Under
`conversation`, a run with no `conversation_id` searches nothing and the tool says
so, rather than falling back to every other unlabelled run.

**Type:** `SearchScope` **Default:** `'all'`

Toolset id for the `search_conversation_history` tool.

**Type:** `str`**Default:** `'conversation-search'`

Maximum number of matching excerpts the search tool returns. Must be non-negative.

**Type:** `int`**Default:** `10`

Number of surrounding lines shown around each search match. Must be non-negative.

**Type:** `int`**Default:** `5`

BM25 term-frequency saturation, non-negative. This capability’s default; Lucene’s
`BM25Similarity` uses `1.2`.

**Type:** `float`**Default:** `1.5`

BM25 length-normalization, between `0.0` and `1.0` (Lucene/Elasticsearch default).

**Type:** `float`**Default:** `0.75`

Emit a short instruction telling the model the recall tool exists.

**Type:** `bool`**Default:** `True`

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
Provide the `search_conversation_history` tool over the source.

[`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset)[`AgentDepsT`] | `None`

```
def get_instructions() -> AgentInstructions[AgentDepsT] | None
```
Tell the model the recall tool exists, unless `add_instructions` is false.

`AgentInstructions`[`AgentDepsT`] | `None`

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/conversation-search
