---
type: Web Page
title: Agent Specs | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/agent-spec
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Agent Specs

Agent specs let you define agents declaratively in YAML or JSON — [model](/docs/ai/models/overview), [instructions](/docs/ai/core-concepts/agent#instructions), [capabilities](/docs/ai/capabilities/overview), and all. One line to load, no Python agent construction code required.

This is useful for:

- Separating agent configuration from application code
- Letting non-developers (prompt engineers, domain experts) configure agents
- Storing agent definitions alongside other config files
- Sharing agent configurations across teams or projects

A spec file defines the agent’s configuration in YAML or JSON:

[ Agent.from_file](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_file) loads a spec from a YAML or JSON file and constructs an agent:

[ Agent.from_spec](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_spec) accepts a dict or 

[instance and supports additional keyword arguments that supplement or override the spec:](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec)

`AgentSpec`Keyword arguments interact with spec fields as follows:

- **Scalar fields**(- `model`,- `name`,- `end_strategy`, etc.) — the keyword argument overrides the spec value when provided. For retry budgets, the- `retries`keyword argument overrides the spec’s- `retries`value.
- `instructions`
- `capabilities`
- `model_settings`
- `output_type`- `output_schema`from the spec.

When `deps_type` is passed, [template strings](#template-strings) in the spec’s `instructions`, `description`, and capability arguments are compiled and validated against the deps type at construction time.

For more control over spec loading, use [ AgentSpec.from_file](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec.from_file) to load the spec separately before passing it to 

`Agent.from_spec`.[ TemplateStr](/docs/ai/api/pydantic-ai/template/#pydantic_ai.template.TemplateStr) provides Handlebars-style templates (

`{{variable}}`) that are rendered against the agent’s [dependencies](/docs/ai/core-concepts/dependencies)at runtime. In spec files, strings containing

`{{` are automatically converted to template strings:```
instructions: "You are assisting {{name}}, who is a {{role}}."
```
Template variables are resolved from the fields of the `deps` object. When a `deps_type` (or [ deps_schema](#deps_schema)) is provided, template variable names are validated at construction time.

In Python code, [ TemplateStr](/docs/ai/api/pydantic-ai/template/#pydantic_ai.template.TemplateStr) can be used explicitly, but a callable with 

[is generally preferred for IDE autocomplete and type checking:](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`Capabilities in specs support three forms:

- `'MyCapability'`— no arguments, calls- `MyCapability.from_spec()`
- `{'MyCapability': value}`— single positional argument, calls- `MyCapability.from_spec(value)`
- `{'MyCapability': {key: value, ...}}`— keyword arguments, calls- `MyCapability.from_spec(**kwargs)`

See [Publishing capabilities](/docs/ai/capabilities/custom#publishing-capabilities) for how to make custom capabilities work with agent specs.

The [ AgentSpec](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec) model represents the full spec structure:

| Field | Type | Description | 
|---|---|---|
| `model` | `str` | [Model](/docs/ai/models/overview)name (required) | 
| `name` | `str | None` | Agent name | 
| `description` | `str | None` | Agent description (supports [templates](#template-strings)) | 
| `instructions` | `str | list[str] | None` | [Instructions](/docs/ai/core-concepts/agent#instructions)(supports[templates](#template-strings)) | 
| `model_settings` | `dict | None` | [Model settings](/docs/ai/core-concepts/agent#model-run-settings) | 
| `capabilities` | `list` | [Capabilities](/docs/ai/capabilities/overview)(see[spec syntax](#capability-spec-syntax)) | 
| `deps_schema` | `dict | None` | JSON Schema for [template string](#template-strings)validation (see below) | 
| `output_schema` | `dict | None` | JSON Schema for [structured output](/docs/ai/core-concepts/output)(see below) | 
| `retries` | `int | AgentRetries | None` | Retry budgets for [tools](/docs/ai/tools-toolsets/tools-advanced#tool-retries)and[output validation](/docs/ai/core-concepts/output#output-validator-functions). Pass an integer to use the same budget for both, orto configure them separately.`AgentRetries` | 
| `end_strategy` | `EndStrategy` | When to stop ( `'early'`,`'graceful'`, or`'exhaustive'`) | 
| `tool_timeout` | `float | None` | Default [tool](/docs/ai/tools-toolsets/tools)timeout in seconds | 
| `instrument` | `bool | None` | Enable [Logfire](/docs/ai/integrations/logfire)instrumentation | 
| `metadata` | `dict | None` | Agent [metadata](/docs/ai/core-concepts/agent#run-metadata) | 

When loading a spec file without a Python `deps_type`, `deps_schema` provides a JSON Schema that validates [template string](#template-strings) variable names at construction time. It does **not** validate the actual deps object at runtime — it only ensures that template variables like `{{user_name}}` correspond to properties defined in the schema.

When provided (and no `output_type` keyword argument is passed to `from_spec`), `output_schema` defines the structure the model should produce as its final output. Under the hood, it creates a [ StructuredDict](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.StructuredDict) output type: the JSON Schema is sent to the model API so the model knows what structure to produce, and the response is returned as a 

`dict[str, Any]`.[ AgentSpec.to_file](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec.to_file) saves a spec to YAML or JSON and optionally generates a companion JSON Schema file for editor autocompletion:

The generated JSON Schema file enables autocompletion and validation in editors that support the [YAML Language Server](https://github.com/redhat-developer/yaml-language-server) protocol. Pass `schema_path=None` to skip schema generation.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/agent-spec
