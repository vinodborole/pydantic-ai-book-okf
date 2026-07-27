---
type: Web Page
title: Resolve Model ID | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/resolve-model-id
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Resolve Model ID

[ ResolveModelId](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ResolveModelId) is a 

[capability](/docs/ai/capabilities/overview)that turns application-specific model IDs into

[instances. The resolver can use run dependencies to look up tenant-specific providers, credentials, or model registries:](/docs/ai/api/models/base/#pydantic_ai.models.Model)

`Model`The resolver may be synchronous or asynchronous. Its full callable signature is
`(ModelResolutionContext[Deps], str) -> Model | None | Awaitable[Model | None]`.
The convenience capability adapts both forms to the asynchronous
[ resolve_model_id()](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.resolve_model_id) hook.

Resolvers form a chain in capability order: the first non-`None` result wins, and Pydantic AI falls back to normal model inference if every resolver returns `None`. See [Resolving model IDs](/docs/ai/capabilities/custom#resolving-model-ids) to implement the hook in a custom capability and understand when each resolver tree is used.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/resolve-model-id
