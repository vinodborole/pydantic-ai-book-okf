---
type: Web Page
title: Compaction | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/compaction
timestamp: '2026-08-10T07:48:56.025339+00:00'
---

# Compaction

As a conversation grows, its message history can approach the model’s context window. *Compaction* keeps it in check by shrinking older messages — trimming, clearing, or summarizing them — while preserving recent context and tool-call integrity. Pydantic AI supports this at several levels, from provider-native APIs to model-agnostic history editing.

Some providers expose a built-in compaction API that runs on their side. Pydantic AI wraps these as [capabilities](/docs/ai/capabilities/overview):

| Provider | Capability | Details | 
|---|---|---|
| OpenAI Responses API | [`OpenAICompaction`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAICompaction) | [OpenAI compaction](/docs/ai/models/openai#message-compaction) | 
| Anthropic | [`AnthropicCompaction`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicCompaction) | [Anthropic compaction](/docs/ai/models/anthropic#message-compaction) | 

Each uses the corresponding provider API, so it’s only available on that provider.

Pydantic AI treats a compaction part as a visibility boundary: the model starts anew from that point for derived tool state. Tool discoveries and on-demand capability loads before the boundary reset, so their tools are hidden again until searched for or loaded after the boundary. Searchable tools remain in the corpus and all registered tools remain callable if the model emits a valid call, even when their earlier schema or reveal evidence is no longer visible to the model. Capability and toolset authors should apply the same rule to their own derived state: compute anything the model needs to have seen — announcements, disclosures, catalogs — from [`post_compaction_window`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.post_compaction_window) rather than remembering it in instance attributes, so it self-heals when compaction replaces the history that carried it.

To compact on any model, edit the message history yourself with a [history processor](/docs/ai/core-concepts/message-history#processing-message-history) wrapped as a [`ProcessHistory`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessHistory) capability — this works with every provider. Common patterns:

- [Keep only recent messages](/docs/ai/core-concepts/message-history#keep-only-recent-messages) — a zero-cost sliding window over the most recent turns.
- [Summarize old messages](/docs/ai/core-concepts/message-history#summarize-old-messages) — use a (cheaper) model to condense older messages into a summary.

[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) packages a menu of ready-made, model-agnostic [compaction strategies](https://pydantic.dev/docs/ai/harness/compaction/): mostly zero-LLM history editing — sliding-window trimming, clearing old tool results, deduplicating repeated file reads, clamping oversized message parts — plus LLM summarization for when that’s not enough, and a `TieredCompaction` orchestrator (the recommended default) that escalates from cheap to expensive strategies only as far as needed to fit the target.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/compaction
