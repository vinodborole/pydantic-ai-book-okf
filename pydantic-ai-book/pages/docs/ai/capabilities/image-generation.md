---
type: Web Page
title: Image Generation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/image-generation
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Image Generation

The `ImageGeneration`[capability](/docs/ai/capabilities/overview) lets your agent generate images. Like all [provider-adaptive tools](/docs/ai/capabilities/overview#provider-adaptive-tools), it uses the provider’s native image generation when available, with an optional subagent fallback for other models.

[ ImageGeneration](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) defaults to native-only. Backed by 

[on the native side (see](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool)

`ImageGenerationTool`[Image Generation Tool](/docs/ai/tools-toolsets/native-tools#image-generation-tool)for provider support and configuration) — pass

`native=ImageGenerationTool(...)` directly for full control.For the local side, pass `fallback_model='…'` to delegate unsupported requests to a subagent running an image-generation-capable model (e.g. `openai-responses:gpt-5.4`), or `local=` with any callable, [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or 

[for a custom generator.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/image-generation
