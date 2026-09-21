---
type: Web Page
title: Function Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/tools
timestamp: '2026-09-21T12:25:24.826293+00:00'
---

# Function Tools

Function tools provide a mechanism for models to perform actions and retrieve extra information to help them generate a response.

They’re useful when you want to enable the model to take some action and use the result, when it is impractical or impossible to put all the context an agent might need into the instructions, or when you want to make agents’ behavior more deterministic or reliable by deferring some of the logic required to generate a response to another (not necessarily AI-powered) tool.

If you want a model to be able to call a function as its final action, without the result being sent back to the model, you can use an [output function](/docs/ai/core-concepts/output/#output-functions) instead.

There are a number of ways to register tools with an agent:

- via the [`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator — for tools that need access to the agent[context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- via the [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain) decorator — for tools that do not need access to the agent[context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- via the [`tools`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) keyword argument to`Agent` which can take either plain functions, or instances of[`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool)

For more advanced use cases, the [toolsets](/docs/ai/tools-toolsets/toolsets/) feature lets you manage collections of tools (built by you or provided by an [MCP server](/docs/ai/mcp/client/) or other [third party](/docs/ai/tools-toolsets/third-party-tools/#third-party-tools)) and register them with an agent in one go via the [`toolsets`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) keyword argument to `Agent`. Internally, all `tools` and `toolsets` are gathered into a single [combined toolset](/docs/ai/tools-toolsets/toolsets/#combining-toolsets) that’s made available to the model.

`@agent.tool` is considered the default decorator since in the majority of cases tools will need access to the agent [context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext).

Here’s an example using both:

This is a pretty simple task, so we can use the fast and cheap Gemini flash model.

We pass the user's name as a string dependency to keep the example simple.

This tool doesn't need any context, it just returns a random number. You could probably use dynamic instructions in this case.

This tool needs the player's name, so it uses `RunContext` to access dependencies which are just the player's name in this case.

Run the agent, passing the player's name as the dependency.

*(This example is complete, it can be run “as is”)*

Let’s print the messages from that game to see what happened:

We can represent this with a diagram:

```
sequenceDiagram
    participant Agent
    participant LLM
    Note over Agent: Send prompts
    Agent ->> LLM: System: "You're a dice game..."<br>User: "My guess is 4"
    activate LLM
    Note over LLM: LLM decides to use<br>a tool
    LLM ->> Agent: Call tool<br>roll_dice()
    deactivate LLM
    activate Agent
    Note over Agent: Rolls a six-sided die
    Agent -->> LLM: ToolReturn<br>"4"
    deactivate Agent
    activate LLM
    Note over LLM: LLM decides to use<br>another tool
    LLM ->> Agent: Call tool<br>get_player_name()
    deactivate LLM
    activate Agent
    Note over Agent: Retrieves player name
    Agent -->> LLM: ToolReturn<br>"Anne"
    deactivate Agent
    activate LLM
    Note over LLM: LLM constructs final response
    LLM ->> Agent: ModelResponse<br>"Congratulations Anne, ..."
    deactivate LLM
    Note over Agent: Game session complete
```
As well as using the decorators, we can register tools via the `tools` argument to the [`Agent` constructor](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__). This is useful when you want to reuse tools, and can also give more fine-grained control over the tools.

The simplest way to register tools via the `Agent` constructor is to pass a list of functions, the function signature is inspected to determine if the tool takes [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext).

*(This example is complete, it can be run “as is”)*

Tools can return anything that Pydantic can serialize to JSON. For advanced output options including multi-modal content and metadata, see [Advanced Tool Features](/docs/ai/tools-toolsets/tools-advanced/#function-tool-output).

Function parameters are extracted from the function signature, and all parameters except `RunContext` are used to build the schema for that tool call.

Even better, Pydantic AI extracts the docstring from functions and (thanks to [griffe](https://mkdocstrings.github.io/griffe/)) extracts parameter descriptions from the docstring and adds them to the schema.

[Griffe supports](https://mkdocstrings.github.io/griffe/reference/docstrings/#docstrings) extracting parameter descriptions from `google`, `numpy`, and `sphinx` style docstrings. Pydantic AI will infer the format to use based on the docstring, but you can explicitly set it using [`docstring_format`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DocstringFormat). You can also enforce parameter requirements by setting `require_parameter_descriptions=True`. This will raise a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) if a parameter description is missing.

Three parts of the docstring reach the model: the leading description, the parameter descriptions, and the
first entry of the returns section. Other sections Griffe can parse, such as `Raises`, `Examples`, `Notes`,
`Warnings` and `Yields`, are dropped, so anything the model needs to act on belongs in the
leading description, a parameter description, or the first returns entry.

To demonstrate a tool’s schema, here we use [`FunctionModel`](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel) to print the schema a model would receive:

*(This example is complete, it can be run “as is”)*

If a tool has a single parameter that can be represented as an object in JSON schema (e.g. dataclass, TypedDict, pydantic model), the schema for the tool is simplified to be just that object.

Here’s an example where we use [`TestModel.last_model_request_parameters`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel.last_model_request_parameters) to inspect the tool schema that would be passed to the model.

*(This example is complete, it can be run “as is”)*

A docstring written under a parameter, a field, or an enum member is the natural place to say what it means, but not every one of them reaches the model:

| Written under | Describes | Sent to the model | 
|---|---|---|
| a tool or [output function](/docs/ai/core-concepts/output/#output-functions) | that tool | always | 
| an `Args:` entry in its docstring | that parameter | always | 
| a class used as a parameter or [output type](/docs/ai/core-concepts/output/) | that object | always | 
| a field of a `dataclass` or`TypedDict` | that field | as a function’s parameter | 
| a field of a Pydantic model or `pydantic.dataclasses.dataclass` | that field | with `use_attribute_docstrings` on the class | 
| an `Enum` member | that option | with [`UseEnumMemberDocstrings`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.UseEnumMemberDocstrings) | 

The schema Pydantic AI builds from a function signature — a tool or an output function — turns on Pydantic’s
[`use_attribute_docstrings`](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.use_attribute_docstrings),
and that reaches every type below it that doesn’t bring a config of its own, at any depth. A Pydantic model and
a `pydantic.dataclasses.dataclass` do bring one, which wins, so they ignore it — for their own fields and for
anything nested inside them. Set it on the class itself to opt in wherever it’s used, including as an
[output type](/docs/ai/core-concepts/output/), which is built from the type rather than from a signature and so never inherits it:

A description written as `Field(description=...)` needs no config and takes precedence over the docstring.

An `Enum` parameter reaches the model as the list of its values, which says what the options are but not what
they mean. Mix [`UseEnumMemberDocstrings`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.UseEnumMemberDocstrings) into the enum to send the
docstring written under each member as that option’s description. Such an enum renders as `anyOf` of `const`
values instead of a plain `enum` list:

*(This example is complete, it can be run “as is”)*

`Enum` is the one case that needs a mix-in rather than a config flag: it has no `model_config` to carry one, and
cannot carry a plain class attribute either — annotated or not, any assigned value becomes a member — so a base
class is the only marker left. Without it the docstrings are ignored and the schema is exactly the one Pydantic
generates on its own, so opting an enum in is the only thing that changes what a model sees.

Members without a docstring keep a bare `const`, and a docstring under an alias (`urgent = 'high'` beside
`high = 'high'`) describes the option it was written for. A `Literal` has nowhere to write a docstring, so it is
unaffected.

Wherever Pydantic AI describes an opted-in enum to a model — a tool parameter, an [output type](/docs/ai/core-concepts/output/), or a
field of a model of your own — the descriptions come along.

The docstrings are read from the enum’s source, so an enum built at run time has none. When the options
themselves are only known once the run is under way, use [`Choices()`](/docs/ai/core-concepts/output/#choices), which describes a
mapping built where you build it and emits exactly this `anyOf`-of-`const`s schema.

A tool can push extra messages into the conversation via
[`RunContext.enqueue`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.enqueue) — useful when a tool wants
to add follow-up context, redirect the agent’s plan, or surface an event the model
should react to. See [Injecting messages mid-run](/docs/ai/core-concepts/message-history/#injecting-messages-mid-run)
for the full pattern.

For more tool features and integrations, see:

- [Advanced Tool Features](/docs/ai/tools-toolsets/tools-advanced/) - Custom schemas, dynamic tools, tool execution and retries
- [Toolsets](/docs/ai/tools-toolsets/toolsets/) - Managing collections of tools
- [Native Tools](/docs/ai/tools-toolsets/native-tools/) - Native tools provided by LLM providers
- [Common Tools](/docs/ai/tools-toolsets/common-tools/) - Ready-to-use tool implementations
- [Third-Party Tools](/docs/ai/tools-toolsets/third-party-tools/) - Integrations with MCP, LangChain, and other tool libraries
- [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools/) - Tools requiring approval or external execution

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/tools
