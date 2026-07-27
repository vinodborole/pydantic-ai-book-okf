---
type: Web Page
title: Select Model | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/select-model
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Select Model

[ SelectModel](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SelectModel) is a 

[capability](/docs/ai/capabilities/overview)that chooses a model from run dependencies, message history, usage, or the current step. The selector is first evaluated during run setup, so the agent does not need a constructor model:

`SelectModel` always receives a callable, which is evaluated before each new logical model request step. The callable may be synchronous or asynchronous. When it returns the same model ID on multiple steps, the resolved model/provider instance is reused for the rest of that run. Provider-side continuation polling within the same step remains pinned to the selected model. See [Selecting the model](/docs/ai/capabilities/custom#selecting-the-model) to implement the hook in a custom capability and for precedence and lifecycle details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/select-model
