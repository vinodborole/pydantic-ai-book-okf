---
type: Web Page
title: Image Generation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/image-generation
timestamp: '2026-09-07T12:01:58.556264+00:00'
---

# Image Generation

The `ImageGeneration`[capability](/docs/ai/capabilities/overview/) lets your agent generate images. Like all [provider-adaptive tools](/docs/ai/capabilities/overview/#provider-adaptive-tools), it uses the provider’s native image generation when available, with an optional subagent fallback for other models.

[`ImageGeneration`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) defaults to native-only. Backed by [`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool) on the native side (see [Image Generation Tool](/docs/ai/tools-toolsets/native-tools/#image-generation-tool) for provider support and configuration) — pass `native=ImageGenerationTool(...)` directly for full control, or a callable taking [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) that returns an [`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool) or `None` — [`ImageGenerationNativeTool`](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.image_generation.ImageGenerationNativeTool) is the type the `fallback_subagent_model` subagent accepts. A callable resolves on each model request, and again when the subagent runs — so keep it free of one-shot side effects. Both resolutions belong to the same run and carry the same `deps`, but they do not share a [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext): the subagent resolves from its own tool call, so `tool_call_id` and `tool_name` name that call instead of being `None`, and `messages` holds the run so far. Read `ctx.deps` for configuration that has to match across both. On the subagent, capability-level fields override the factory result. See [Dynamic Configuration](/docs/ai/tools-toolsets/native-tools/#dynamic-configuration).

For the local side, pass `fallback_subagent_model='…'` to delegate unsupported requests to a subagent running an image-generation-capable model (e.g. `openai-responses:gpt-5.4`), or `local=` with any callable, [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) for a custom generator.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/image-generation
