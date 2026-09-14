---
type: Web Page
title: Image Generation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/image-generation
timestamp: '2026-09-14T12:17:54.595402+00:00'
---

# Image Generation

The `ImageGeneration`[capability](/docs/ai/capabilities/overview/) lets an agent decide when to
generate an image. It prefers the conversational model provider’s native image-generation tool and can fall back to a
dedicated image model through the [direct image-generation API](/docs/ai/guides/image-generation/).

`ImageGeneration()` is native-only by default. Add a fallback that generates through the image API — without creating
another agent — with one of two fields, and which one you use follows what you have: an
[`ImageGenerator`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerator) carries settings of its own, so it goes on `local` beside the
other implementations you supply; a bare [`ImageGenerationModel`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerationModel) or a
`'provider:model'` name goes on `fallback_image_model`. Either way the direct model generates through the image API,
where `fallback_subagent_model` runs a conversational model in a subagent, and it is reached when the agent’s model has no
native image generation — or on every call, with `native=False`. `ImageGeneration` has no bundled local strategy, so
`local` otherwise takes a tool of your own.

Two of the fields name an image model and they are not interchangeable: `image_model` configures the provider’s native
tool and is unprefixed (`image_model='gpt-image-2'`), while `fallback_image_model` selects the direct model and carries
its provider:

The portable `dimensions` and `aspect_ratio` capability settings override defaults on an explicit generator.
Only the direct generator can apply `dimensions`, and only it can apply the aspect ratios the native tool does not
share, so pass `native=False` when you need either to be guaranteed: with the default `native=True` a model that
generates images natively takes the native path, which has no equivalent for them, and the request warns that the
settings went unapplied. Native-tool-only settings such as
`quality` and `output_format` do not apply to a direct fallback; configure their
provider-prefixed equivalents on the generator. `action='edit'` and `image_model` do not apply either: the direct
fallback raises [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) for `action='edit'`, because the `generate_image` tool
receives no reference images, and ignores `image_model` with a warning, because the generator already names the image
model it generates with.
`native=False` makes the direct generator the only path, so both of those land at construction — the dropped settings
as a warning and `action='edit'` as the error; with native enabled the native tool still carries them, so a request
that routes to the direct generator instead is what warns that they went unapplied, and what raises for
`action='edit'`. The direct generator must return exactly one generated
[`BinaryImage`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryImage); use
[`ImageGenerator`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerator) directly for multiple images or reference-image editing.

[`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool) is the native implementation (see
[Image Generation Tool](/docs/ai/tools-toolsets/native-tools/#image-generation-tool) for provider support and configuration). Pass an
explicit instance through `native=ImageGenerationTool(...)` when you need its full provider-native configuration, or a
callable taking [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) that returns an `ImageGenerationTool` or `None` for
[dynamic configuration](/docs/ai/tools-toolsets/native-tools/#dynamic-configuration). A callable resolves on each model request and again
when the `fallback_subagent_model` subagent runs. Both resolutions receive the same `deps`, but the subagent has its own
`RunContext`; use `ctx.deps` for configuration that must match across both. Capability-level fields override the
factory result on the subagent.

A static native instance’s `aspect_ratio` reaches whichever fallback you configured — the `fallback_subagent_model` subagent or
the direct generator — while a capability-level `aspect_ratio` takes precedence over it, as does a
capability-level `dimensions`, which is the same geometry spelled differently and cannot be combined with it. Only
inheritance yields that way: setting both `dimensions` and `aspect_ratio` on the capability itself raises
[`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) at construction, once a direct generator is configured to apply
them.

Instrumentation is per generator, not per agent: the agent-level
[`Instrumentation`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Instrumentation) capability does not reach the direct generator,
so a run records no `image_generation` span unless the generator carries its own. Pass `instrument=` when you construct
it, or switch it on globally with
[`ImageGenerator.instrument_all()`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerator.instrument_all):

See [Instrumentation](/docs/ai/guides/image-generation/#instrumentation) for what those spans carry.

Two built-in mechanisms cover a model that does not generate images natively:

- **Direct model** :`local=` with an[`ImageGenerator`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerator) , or`fallback_image_model=` with an image model name or[`ImageGenerationModel`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerationModel) . The tool call is a
single image API call, with no extra agent run, and it applies the portable`dimensions` and`aspect_ratio` settings.
Reach for it when the geometry or the choice of image model is yours to make.
- **Subagent** :`fallback_subagent_model=` with a conversational model that generates images natively. The tool call runs an
additional agent whose native[`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool) produces the
image. Reach for it when you want that model’s native tool semantics and the settings the native tool carries.

A `local=` callable, `Tool`, or toolset of your own replaces both with an implementation you write. The three fields are
alternatives: stating more than one raises [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError).

The subagent speaks the native tool’s geometry vocabulary. Direct-only values such as `dimensions`, arbitrary GPT
Image 2 sizes, and additional aspect ratios are ignored with a warning. Use `native=False` with a direct generator —
`local=ImageGenerator(...)` or `fallback_image_model='provider:image-model'` — to apply the
[direct geometry settings](/docs/ai/guides/image-generation/#output-geometry).

A provider content block becomes a retry prompt on either local path, so the model that wrote the prompt gets to
rephrase it rather than failing the run. Other failures differ between the two: the subagent also turns an
[`UnexpectedModelBehavior`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior) from its run into a retry prompt, while the
direct model raises it. Using
[`ImageGenerator`](/docs/ai/api/pydantic-ai/images/#pydantic_ai.images.ImageGenerator) on its own is unaffected — it raises
[`ContentFilterError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ContentFilterError). See
[error handling](/docs/ai/guides/image-generation/#error-handling) for the direct API’s exceptions.

Direct model names such as `fallback_image_model='openai:gpt-image-1.5'` can be represented in JSON or YAML agent
specs. Runtime objects accepted by the Python constructor — `ImageGenerationModel` on `fallback_image_model`, and the
`ImageGenerator`, `Tool`, toolsets and callables `local` takes — are not serializable and must be configured in Python.
[`from_spec()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration.from_spec) keeps that serializable subset explicit while
exposing the same setting names. Write `dimensions` as the two-item array used by JSON and YAML; Pydantic AI converts
it to the `(width, height)` tuple used by the Python API:

```
model: anthropic:claude-sonnet-4-6
capabilities:
  - ImageGeneration:
      native: false
      fallback_image_model: openai:gpt-image-2
      dimensions: [1280, 720]
```

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/image-generation
