---
type: Web Page
title: Raise Content Filter Error | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/raise-content-filter-error
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Raise Content Filter Error

[`RaiseContentFilterError`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.RaiseContentFilterError) is a [capability](/docs/ai/capabilities/overview) that opts into treating any model response with `finish_reason='content_filter'` as a [`ContentFilterError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ContentFilterError), even when the provider returns partial text or refusal text:

*(This example is complete, it can be run “as is”)*

By default, Pydantic AI only raises [`ContentFilterError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ContentFilterError) when a `content_filter` response is *empty*: if the provider returns partial text or refusal text alongside `finish_reason='content_filter'`, that text becomes ordinary agent output and no error is raised (see [finish reason handling](/docs/ai/models/overview#finish-reason-example)). This capability extends the check to *every* `content_filter` response, so partial and refusal text raise too. When it raises, the full [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) is serialized into [`ContentFilterError.body`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior.body) so the partial text remains inspectable.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/raise-content-filter-error
