---
type: Web Page
title: Include Tool Return Schemas | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/include-tool-return-schemas
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Include Tool Return Schemas

[ IncludeToolReturnSchemas](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.IncludeToolReturnSchemas) is a 

[capability](/docs/ai/capabilities/overview)that includes return type schemas in tool definitions sent to the model. For models that natively support return schemas (e.g. Google Gemini), the schema is passed as a structured field in the API request. For other models, it is injected into the tool description as JSON text.

*(This example is complete, it can be run “as is”)*

Use the `tools` parameter to select which tools should include return schemas. It accepts a list of tool names, a metadata dict for matching, or a callable predicate:

*(This example is complete, it can be run “as is”)*

The same effect can be achieved at the toolset level using [ .include_return_schemas()](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.include_return_schemas) — see 

[toolset composition](/docs/ai/tools-toolsets/toolsets#including-return-schemas).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/include-tool-return-schemas
