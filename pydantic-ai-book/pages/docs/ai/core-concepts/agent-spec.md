---
type: Web Page
title: Agent Specs | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/agent-spec
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Agent Specs

Agent specs let you define agents declaratively in YAML or JSON — model, instructions, capabilities, and all. One line to load, no Python agent construction code required.

This is useful for:

- Separating agent configuration from application code
- Letting non-developers (prompt engineers, domain experts) configure agents
- Storing agent definitions alongside other config files
- Sharing agent configurations across teams or projects

A spec file defines the agent’s configuration in YAML or JSON:

`Agent.from_file` loads a spec from a YAML or JSON file and constructs an agent:

`Agent.from_spec` accepts a dict or `AgentSpec` instance and supports additional keyword arguments that supplement or override the spec:

Keyword arguments interact with spec fields as follows:

- **Scalar fields**(- `model`,- `name`,- `end_strategy`, etc.) — the keyword argument overrides the spec value when provided. For retry budgets, the- `retries`keyword argument overrides the spec’s- `retries`value.
- `instructions`
- `capabilities`
- `model_settings`
- `output_type`- `output_schema`from the spec.

When `deps_type` is passed, template strings in the spec’s `instructions`, `description`, and capability arguments are compiled and validated against the deps type at construction time.

For more control over spec loading, use `AgentSpec.from_file` to load the spec separately before passing it to `Agent.from_spec`.

`TemplateStr` provides Handlebars-style templates (`{{variable}}`) that are rendered against the agent’s dependencies at runtime. In spec files, strings containing `{{` are automatically converted to template strings:

```
instructions: "You are assisting {{name}}, who is a {{role}}."
```
Template variables are resolved from the fields of the `deps` object. When a `deps_type` (or `deps_schema`) is provided, template variable names are validated at construction time.

In Python code, `TemplateStr` can be used explicitly, but a callable with `RunContext` is generally preferred for IDE autocomplete and type checking:

Capabilities in specs support three forms:

- `'MyCapability'`— no arguments, calls- `MyCapability.from_spec()`
- `{'MyCapability': value}`— single positional argument, calls- `MyCapability.from_spec(value)`
- `{'MyCapability': {key: value, ...}}`— keyword arguments, calls- `MyCapability.from_spec(**kwargs)`

See Publishing capabilities for how to make custom capabilities work with agent specs.

The `AgentSpec` model represents the full spec structure:

| Field | Type | Description | 
|---|---|---|
| `model` | `str` | Model name (required) | 
| `name` | `str | None` | Agent name | 
| `description` | `str | None` | Agent description (supports templates) | 
| `instructions` | `str | list[str] | None` | Instructions (supports templates) | 
| `model_settings` | `dict | None` | Model settings | 
| `capabilities` | `list` | Capabilities (see spec syntax) | 
| `deps_schema` | `dict | None` | JSON Schema for template string validation (see below) | 
| `output_schema` | `dict | None` | JSON Schema for structured output (see below) | 
| `retries` | `int | AgentRetries | None` | Retry budgets for tools and output validation. Pass an integer to use the same budget for both, or `AgentRetries`to configure them separately. | 
| `end_strategy` | `EndStrategy` | When to stop ( `'early'`,`'graceful'`, or`'exhaustive'`) | 
| `tool_timeout` | `float | None` | Default tool timeout in seconds | 
| `instrument` | `bool | None` | Enable Logfire instrumentation | 
| `metadata` | `dict | None` | Agent metadata | 

When loading a spec file without a Python `deps_type`, `deps_schema` provides a JSON Schema that validates template string variable names at construction time. It does **not** validate the actual deps object at runtime — it only ensures that template variables like `{{user_name}}` correspond to properties defined in the schema.

When provided (and no `output_type` keyword argument is passed to `from_spec`), `output_schema` defines the structure the model should produce as its final output. Under the hood, it creates a `StructuredDict` output type: the JSON Schema is sent to the model API so the model knows what structure to produce, and the response is returned as a `dict[str, Any]`.

`AgentSpec.to_file` saves a spec to YAML or JSON and optionally generates a companion JSON Schema file for editor autocompletion:

The generated JSON Schema file enables autocompletion and validation in editors that support the YAML Language Server protocol. Pass `schema_path=None` to skip schema generation.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/agent-spec
