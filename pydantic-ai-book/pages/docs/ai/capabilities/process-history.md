---
type: Web Page
title: Process History | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/process-history
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Process History

[ ProcessHistory](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessHistory) is a 

[capability](/docs/ai/capabilities/overview)that wraps a

[history processor](/docs/ai/core-concepts/message-history#processing-message-history): a function that receives the message history before each model request and returns the (possibly modified) list of messages to send. Use it to trim old turns, redact sensitive content, or summarize long conversations:

Keep only the five most recent messages. In practice you'll want to keep the first request too, so the system prompt survives — see [Processing Message History](/docs/ai/core-concepts/message-history#processing-message-history) for complete patterns.

The processor may be sync or async, and may optionally take a [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) as its first argument to access dependencies and run state. Multiple 

`ProcessHistory` capabilities apply in registration order. Note that the processed messages *replace*the run’s message history, so make a copy first if you need to keep the original.

`ProcessHistory` is a thin wrapper around the [ before_model_request](/docs/ai/core-concepts/hooks) lifecycle hook — hook that event directly for richer control, like short-circuiting the model call. See 

[Processing Message History](/docs/ai/core-concepts/message-history#processing-message-history)for the full guide, including summarization examples and interactions with

[.](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.new_messages)

`new_messages()`

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/process-history
