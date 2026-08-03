---
type: Web Page
title: Set Tool Metadata | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/set-tool-metadata
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Set Tool Metadata

[`SetToolMetadata`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) is a [capability](/docs/ai/capabilities/overview) that merges metadata key-value pairs onto selected tools. This is useful for tagging tools with configuration that other capabilities or custom logic can inspect:

*(This example is complete, it can be run “as is”)*

The same effect can be achieved at the toolset level using [`.with_metadata()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.with_metadata) — see [toolset composition](/docs/ai/tools-toolsets/toolsets#setting-tool-metadata).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/set-tool-metadata
