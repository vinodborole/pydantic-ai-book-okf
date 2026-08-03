---
type: Web Page
title: Reinject System Prompt | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/reinject-system-prompt
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Reinject System Prompt

[`ReinjectSystemPrompt`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ReinjectSystemPrompt) is a [capability](/docs/ai/capabilities/overview) that ensures the agent’s configured [`system_prompt`](/docs/ai/core-concepts/agent#system-prompts) is at the head of the first [`ModelRequest`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest) on every model request. By default, if any [`SystemPromptPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.SystemPromptPart) is already present in the history, the capability is a no-op (so multi-agent handoff and user-managed system prompts remain authoritative). Set `replace_existing=True` to instead strip any existing `SystemPromptPart`s before prepending the agent’s configured prompt — useful when the history comes from an untrusted source and the server’s prompt must win.

Useful when `message_history` comes from a source that doesn’t round-trip system prompts — UI frontends, database persistence layers, conversation compaction pipelines. Without this capability, an agent configured with a `system_prompt` will silently run without it if the history doesn’t already include one.

*(This example is complete, it can be run “as is”)*

The [UI adapters](/docs/ai/integrations/ui/ag-ui) (AG-UI, Vercel AI) automatically add this capability with `replace_existing=True` in their `manage_system_prompt='server'` mode.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/reinject-system-prompt
