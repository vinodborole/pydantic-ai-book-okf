---
type: Web Page
title: Reinject System Prompt | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/reinject-system-prompt
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Reinject System Prompt

[ ReinjectSystemPrompt](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ReinjectSystemPrompt) is a 

[capability](/docs/ai/capabilities/overview)that ensures the agent’s configured

[is at the head of the first](/docs/ai/core-concepts/agent#system-prompts)

`system_prompt`[on every model request. By default, if any](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)

`ModelRequest`[is already present in the history, the capability is a no-op (so multi-agent handoff and user-managed system prompts remain authoritative). Set](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.SystemPromptPart)

`SystemPromptPart``replace_existing=True` to instead strip any existing `SystemPromptPart`s before prepending the agent’s configured prompt — useful when the history comes from an untrusted source and the server’s prompt must win.Useful when `message_history` comes from a source that doesn’t round-trip system prompts — UI frontends, database persistence layers, conversation compaction pipelines. Without this capability, an agent configured with a `system_prompt` will silently run without it if the history doesn’t already include one.

*(This example is complete, it can be run “as is”)*

The [UI adapters](/docs/ai/integrations/ui/ag-ui) (AG-UI, Vercel AI) automatically add this capability with `replace_existing=True` in their `manage_system_prompt='server'` mode.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/reinject-system-prompt
