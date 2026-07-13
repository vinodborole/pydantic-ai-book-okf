---
type: Web Page
title: Agents | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/agent
timestamp: '2026-07-13T09:36:10.292247+00:00'
---

# Agents

Agents are Pydantic AI’s primary interface for interacting with LLMs.

In some use cases a single Agent will control an entire application or component, but multiple agents can also interact to embody more complex workflows.

The [ Agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) class has full API documentation, but conceptually you can think of an agent as a container for:

| Component | Description | 
|---|---|
| [Instructions](#instructions) | A set of instructions for the LLM written by the developer. | 
| [Function tool(s)](/docs/ai/tools-toolsets/tools)and[toolsets](/docs/ai/tools-toolsets/toolsets) | Functions that the LLM may call to get information while generating a response. | 
| [Structured output type](/docs/ai/core-concepts/output) | The structured datatype the LLM must return at the end of a run, if specified. | 
| [Dependency type constraint](/docs/ai/core-concepts/dependencies) | Dynamic instructions functions, tools, and output functions may all use dependencies when they’re run. | 
| [LLM model](/docs/ai/models/base) | Optional default LLM model associated with the agent. Can also be specified when running the agent. | 
| [Model Settings](#additional-configuration) | Optional default model settings to help fine tune requests. Can also be specified when running the agent. | 
| [Capabilities](/docs/ai/core-concepts/capabilities) | Reusable bundles of tools, hooks, instructions, and model settings that extend agent behavior. | 

While each of these can be configured individually, [capabilities](/docs/ai/core-concepts/capabilities) let you bundle related behavior into reusable units that are easier to compose, share, and [load from configuration files](/docs/ai/core-concepts/agent-spec).

In typing terms, agents are generic in their dependency and output types, e.g., an agent which required dependencies of type `Foobar` and produced outputs of type `list[str]` would have type `Agent[Foobar, list[str]]`. In practice, you shouldn’t need to care about this, it should just mean your IDE can tell you when you have the right type, and if you choose to use [static type checking](#static-type-checking) it should work well with Pydantic AI.

Here’s a toy example of an agent that simulates a roulette wheel:

Create an agent, which expects an integer dependency and produces a boolean output. This agent will have type `Agent[int, bool]`.

Define a tool that checks if the square is a winner. Here [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is parameterized with the dependency type 

`int`; if you got the dependency type wrong you'd get a typing error.In reality, you might want to use a random number here e.g. `random.randint(0, 36)`.

`result.output` will be a boolean indicating if the square is a winner. Pydantic performs the output validation, and it'll be typed as a `bool` since its type is derived from the `output_type` generic parameter of the agent.

There are five ways to run an agent:

- `agent.run()`- `RunResult`
- `agent.run_sync()`- `RunResult`- `loop.run_until_complete(self.run())`).
- `agent.run_stream()`- `StreamedRunResult`- `agent.run_stream_sync()`- `StreamedRunResultSync`
- `agent.run_stream_events()`- `AgentStreamEvent`s- `AgentRunResultEvent`
- `agent.iter()`- `AgentRun`- `Graph`

Here’s a simple example demonstrating the first four:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

You can also pass messages from previous runs to continue a conversation or provide context, as described in [Messages and Chat History](/docs/ai/core-concepts/message-history).

As shown in the example above, [ run_stream()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) makes it easy to stream the agent’s final output as it comes in.
It also takes an optional 

`event_stream_handler` argument that you can use to gain insight into what is happening during the run before the final output is produced.The example below shows how to stream events and text output. You can also [stream structured output](/docs/ai/core-concepts/output#streaming-structured-output).

These “dangling” tool calls will not be executed unless the agent’s [ end_strategy](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.end_strategy) is set to 

`'graceful'` or `'exhaustive'`, and even then their results will not be sent back to the model as the agent run will already be considered completed. In short, if the model returns both tool calls and text, and the agent’s output type is `str`, **the tool calls will not run**in streaming mode with the default setting.

If you want to always keep running the agent when it performs tool calls, and stream all events from the model’s streaming response and the agent’s execution of tools,
use [ agent.run_stream_events()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events) or 

[instead, as described in the following sections.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.iter)

`agent.iter()`*(This example is complete, it can be run “as is”)*

Like `agent.run_stream()`, [ agent.run()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) takes an optional 

`event_stream_handler`
argument that lets you stream all events from the model’s streaming response and the agent’s execution of tools.
Unlike `run_stream()`, it always runs the agent graph to completion even if text was received ahead of tool calls that looked like it could’ve been the final result.For convenience, a [ agent.run_stream_events()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events) method is also available as a wrapper around 

`run(event_stream_handler=...)`. It is an async context manager that yields an async iterator over [ending with an](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)

`AgentStreamEvent`s[carrying the final run result.](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResultEvent)

`AgentRunResultEvent`*(This example is complete, it can be run “as is”)*

Under the hood, each `Agent` in Pydantic AI uses **pydantic-graph** to manage its execution flow. **pydantic-graph** is a generic, type-centric library for building and running finite state machines in Python. It doesn’t actually depend on Pydantic AI — you can use it standalone for workflows that have nothing to do with GenAI — but Pydantic AI makes use of it to orchestrate the handling of model requests and model responses in an agent’s run.

In many scenarios, you don’t need to worry about pydantic-graph at all; calling `agent.run(...)` simply traverses the underlying graph from start to finish. However, if you need deeper insight or control — for example to inject your own logic at specific stages — Pydantic AI exposes the lower-level iteration process via [ Agent.iter](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter). This method returns an 

[, which you can async-iterate over, or manually drive node-by-node via the](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun)

`AgentRun`[method. Once the agent’s graph returns an](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next)

`next`[, you have the final result along with a detailed history of all steps.](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End)

`End`Here’s an example of using `async for` with `iter` to record each node the agent executes:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

- The `AgentRun`is an async iterator that yields each node (`BaseNode`or`End`) in the flow.
- The run ends when an `End`node is returned.

You can also drive the iteration manually by passing the node you want to run next to the `AgentRun.next(...)` method. This allows you to inspect or modify the node before it executes or skip nodes based on your own logic, and to catch errors in `next()` more easily:

We start by grabbing the first node that will be run in the agent's graph.

The agent run is finished once an `End` node has been produced; instances of `End` cannot be passed to `next`.

When you call `await agent_run.next(node)`, it executes that node in the agent's graph, updates the run's history, and returns the *next* node to run.

You could also inspect or mutate the new `node` here as needed.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

You can retrieve usage statistics (tokens, requests, etc.) at any time from the [ AgentRun](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun) object via 

`agent_run.usage`. This property returns a [object containing the usage data.](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage)

`RunUsage`Once the run finishes, `agent_run.result` becomes an [ AgentRunResult](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) object containing the final output (and related metadata).

Here is an example of streaming an agent run in combination with `async for` iteration:

*(This example is complete, it can be run “as is”)*

Pydantic AI offers a [ UsageLimits](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) structure to help you limit your
usage (tokens, requests, and tool calls) on model runs.

You can apply these settings by passing the `usage_limits` argument to the `run{_sync,_stream}` functions.

Consider the following example, where we limit the number of output tokens:

```
from pydantic_ai import Agent, UsageLimitExceeded, UsageLimits
agent = Agent('anthropic:claude-sonnet-4-6')
result_sync = agent.run_sync(
    'What is the capital of Italy? Answer with just the city.',
    usage_limits=UsageLimits(output_tokens_limit=10),
)
print(result_sync.output)
#> Rome
print(result_sync.usage)
#> RunUsage(input_tokens=62, output_tokens=1, requests=1)
try:
    result_sync = agent.run_sync(
        'What is the capital of Italy? Answer with a paragraph.',
        usage_limits=UsageLimits(output_tokens_limit=10),
    )
except UsageLimitExceeded as e:
    print(e)
    #> Exceeded the output_tokens_limit of 10 (output_tokens=32)
```
Restricting the number of requests can be useful in preventing infinite loops or excessive tool calling:

This tool has the ability to retry 5 times before erroring, simulating a tool that might get stuck in a loop.

This run will error after 3 requests, preventing the infinite tool calling.

If you need a limit on the number of successful tool invocations within a single run, use `tool_calls_limit`:

```
from pydantic_ai import Agent
from pydantic_ai.exceptions import UsageLimitExceeded
from pydantic_ai.usage import UsageLimits
agent = Agent('anthropic:claude-sonnet-4-6')
@agent.tool_plain
def do_work() -> str:
    return 'ok'
try:
    # Allow at most one executed tool call in this run
    agent.run_sync('Please call the tool twice', usage_limits=UsageLimits(tool_calls_limit=1))
except UsageLimitExceeded as e:
    print(e)
    #> The next tool call(s) would exceed the tool_calls_limit of 1 (tool_calls=2).
```
Tools and [capabilities](/docs/ai/core-concepts/capabilities) can read the run’s limits from [ ctx.usage_limits](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.usage_limits) (alongside 

[for usage so far), so a budget-aware tool or capability can disclose or adapt to the remaining budget without being configured with a duplicate copy of the limits. It reflects what the run is already enforcing and is read-only by convention.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.usage)

`ctx.usage`Pydantic AI offers a [ settings.ModelSettings](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) structure to help you fine tune your requests.
This structure allows you to configure common parameters that influence the model’s behavior, such as 

`temperature`, `max_tokens`, `top_k`,
`timeout`, and more.There are three ways to apply these settings, with a clear precedence order:

- **Model-level defaults**- Set when creating a model instance via the- `settings`parameter. These serve as the base defaults for that model.
- **Agent-level defaults**- Set during- `Agent`- `model_settings`argument. These are merged with model defaults, with agent settings taking precedence.
- **Run-time overrides**- Passed to- `run{_sync,_stream}`functions via the- `model_settings`argument. These have the highest priority and are merged with the combined agent and model defaults.

For example, if you’d like to set the `temperature` setting to `0.0` to ensure less random behavior,
you can do the following:

```
from pydantic_ai import Agent, ModelSettings
from pydantic_ai.models.openai import OpenAIChatModel
# 1. Model-level defaults
model = OpenAIChatModel(
    'gpt-5.2',
    settings=ModelSettings(temperature=0.8, max_tokens=500)  # Base defaults
)
# 2. Agent-level defaults (overrides model defaults by merging)
agent = Agent(model, model_settings=ModelSettings(temperature=0.5))
# 3. Run-time overrides (highest priority)
result_sync = agent.run_sync(
    'What is the capital of Italy?',
    model_settings=ModelSettings(temperature=0.0)  # Final temperature: 0.0
)
print(result_sync.output)
#> The capital of Italy is Rome.
```
The final request uses `temperature=0.0` (run-time), `max_tokens=500` (from model), demonstrating how settings merge with run-time taking precedence.

Both agent-level and run-level `model_settings` accept a callable that receives a
[ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns 

[. The callable is invoked before each model request, so settings can vary per step. The current resolved settings so far are available via](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings)

`ModelSettings``ctx.model_settings` inside the callable.Settings are resolved in layers, each merged on top of the previous:

- **Model defaults**(- `model.settings`)
- **Agent-level**(- `Agent(model_settings=...)`)
- **Capability-level**(e.g. from- `Thinking()`- [Capabilities](/docs/ai/core-concepts/capabilities#providing-model-settings))
- **Run-level**(- `agent.run(model_settings=...)`)

Inside a callable, `ctx.model_settings` contains the merged result of all *previous* layers (position-dependent). For example, an agent-level callable sees only model defaults, while a run-level callable sees model defaults + agent-level + capability-level settings. To reset a field set by a previous layer, set it explicitly (e.g. `{'temperature': None}`).

```
from pydantic_ai import Agent, ModelSettings
agent = Agent(
    'test',
    model_settings=lambda ctx: ModelSettings(
        temperature=0.0 if ctx.run_step <= 1 else 0.7,
    ),
)
```
Run metadata lets you tag each agent execution with contextual details (for example, a tenant ID to filter traces and logs)
and read it after completion via [ AgentRun.metadata](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun),

[, or](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult)

`AgentRunResult.metadata`[. The resolved metadata is attached to the](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult)

`StreamedRunResult.metadata`[during the run and, when instrumentation is enabled, added to the run span attributes for observability tools.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`Configure metadata on an [ Agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) or pass it to a run.
Both accept either a static dictionary or a callable that receives the 

[. Metadata is computed (if a callable) and applied when the run starts, then recomputed after a run ends successfully, so it can include end-of-run values. Agent-level metadata and per-run metadata are merged, with per-run values overriding agent-level ones.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`You can limit the number of concurrent agent runs using the `max_concurrency` parameter.
This is useful when you want to prevent overwhelming external resources or enforce rate limits when running many agent instances in parallel.

When the concurrency limit is reached, additional calls to [ agent.run()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) or 

[will wait until a slot becomes available. If you configure](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter)

`agent.iter()``max_queued` and the queue fills up,
a [exception is raised.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ConcurrencyLimitExceeded)

`ConcurrencyLimitExceeded`When instrumentation is enabled, waiting operations appear as “waiting for concurrency” spans with attributes showing queue depth and limits.

If you wish to further customize model behavior, you can use a subclass of [ ModelSettings](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings), like

[, associated with your model of choice.](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModelSettings)

`GoogleModelSettings`For example:

This error is raised because the safety thresholds were exceeded.

An agent **run** might represent an entire conversation — there’s no limit to how many messages can be exchanged in a single run. However, a **conversation** might also be composed of multiple runs, especially if you need to maintain state between separate interactions or API calls.

Here’s an example of a conversation comprised of multiple runs:

Continue the conversation; without `message_history` the model would not know who "his" was referring to.

*(This example is complete, it can be run “as is”)*

Pydantic AI is designed to work well with static type checkers, like mypy and pyright.

In particular, agents are generic in both the type of their dependencies and the type of the outputs they return, so you can use the type hints to ensure you’re using the right types.

Consider the following script with type mistakes:

The agent is defined as expecting an instance of `User` as `deps`.

But here `add_user_name` is defined as taking a `str` as the dependency, not a `User`.

Since the agent is defined as returning a `bool`, this will raise a type error since `foobar` expects `bytes`.

Running `mypy` on this will give the following output:

Running `pyright` would identify the same issues.

System prompts might seem simple at first glance since they’re just strings (or sequences of strings that are concatenated), but crafting the right system prompt is key to getting the model to behave as you want.

Generally, system prompts fall into two categories:

- **Static system prompts**: These are known when writing the code and can be defined via the- `system_prompt`parameter of the- `Agent`constructor
- **Dynamic system prompts**: These depend in some way on context that isn’t known until runtime, and should be defined via functions decorated with- `@agent.system_prompt`

You can add both to a single agent; they’re appended in the order they’re defined at runtime.

Here’s an example using both types of system prompts:

The agent expects a string dependency.

Static system prompt defined at agent creation time.

Dynamic system prompt defined via a decorator with [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), this is called just after 

`run_sync`, not when the agent is created, so can benefit from runtime information like the dependencies used on that run.Another dynamic system prompt, system prompts don't have to have the `RunContext` parameter.

*(This example is complete, it can be run “as is”)*

Instructions are similar to system prompts. The main difference is that when an explicit `message_history` is provided
in a call to `Agent.run` and similar methods, *instructions* from any existing messages in the history are not included
in the request to the model — only the instructions of the *current* agent are included.

You should use:

- `instructions`when you want your request to the model to only include system prompts for the- *current*agent
- `system_prompt`when you want your request to the model to- *retain*the system prompts used in previous requests (possibly made using other agents)

In general, we recommend using `instructions` instead of `system_prompt` unless you have a specific reason to use `system_prompt`.

Instructions, like system prompts, can be specified at different times:

- **Static instructions**: These are known when writing the code and can be defined via the- `instructions`parameter of the- `Agent`constructor
- **Dynamic instructions**: These rely on context that is only available at runtime and should be defined using functions decorated with- `@agent.instructions`- `message_history`is present, dynamic instructions are always reevaluated.
- **Runtime instructions**: These are additional instructions for a specific run that can be passed to one of the- [run methods](#running-agents)using the- `instructions`argument.

All three types of instructions can be added to a single agent, and they are appended in the order they are defined at runtime. Each instruction is internally classified as either **static** (literal strings from the `instructions` parameter) or **dynamic** (from `@agent.instructions` functions, runtime instructions, or [toolset](/docs/ai/tools-toolsets/toolsets) instructions). Static instructions are always sorted before dynamic ones. This ordering enables providers that support prompt caching (like [Anthropic](/docs/ai/models/anthropic#smart-instruction-caching) and [Bedrock](/docs/ai/models/bedrock#prompt-caching)) to cache the stable static prefix while leaving dynamic instructions outside the cache boundary.

Here’s an example using a static instruction as well as dynamic instructions:

The agent expects a string dependency.

Static instructions defined at agent creation time.

Dynamic instructions defined via a decorator with [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext),
this is called just after 

`run_sync`, not when the agent is created, so can benefit from runtime
information like the dependencies used on that run.Another dynamic instruction, instructions don't have to have the `RunContext` parameter.

*(This example is complete, it can be run “as is”)*

Note that returning an empty string will result in no instruction message added.

Instructions can also come from [capabilities](/docs/ai/core-concepts/capabilities) via [ get_instructions()](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions), or from 

[template strings](/docs/ai/core-concepts/agent-spec#template-strings)rendered against the agent’s dependencies.

Validation errors from both function tool parameter validation and [structured output validation](/docs/ai/core-concepts/output#structured-output) can be passed back to the model with a request to retry.

You can also raise [ ModelRetry](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) from within a 

[tool](/docs/ai/tools-toolsets/tools)or

[output function](/docs/ai/core-concepts/output#output-functions)to tell the model it should retry generating a response.

- The default retry count is **1**but can be altered for the[entire agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__)with`retries`or`AgentRetries`[specific tool](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool), or[outputs](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__). The output side of the agent retry budget can also be overridden per run via`agent.run(retries={'output': ...})`and friends.
- You can access the current retry count from within a tool, output validator, or output function via `ctx.retry`

Pydantic AI enforces the output retry budget differently depending on how the model returns its final output:

- **Text output path**(- `output_type=str`, text-only outputs, empty or unusable model responses): a single global budget is shared across the whole run. Each invalid response consumes one unit of the budget; when it’s exhausted, the run raises- `UnexpectedModelBehavior`- `'Exceeded maximum output retries (N)'`.
- **Tool output path**(- `output_type=ToolOutput(...)`- *default per-tool limit*. See- [Tool Output](/docs/ai/core-concepts/output#tool-output)for per-tool overrides via- `ToolOutput(max_retries=N)`

For how the budget appears inside [output validators](/docs/ai/core-concepts/output#output-validator-functions) — including what `ctx.max_retries` and `ctx.retry` reflect on each path — see the [Output validators](/docs/ai/core-concepts/output#output-validator-functions) section.

Tool retries are tracked per tool — see [Tool Execution and Retries](/docs/ai/tools-toolsets/tools-advanced#tool-retries) for the per-tool counter model and the three configuration levels.

Here’s an example:

Agents require a different approach to observability than traditional software. With traditional web endpoints or data pipelines, you can largely predict behavior by reading the code. With agents, this is much harder. The model’s decisions are stochastic, and that stochasticity compounds through the agentic loop as the agent reasons, calls tools, observes results, and reasons again. You need to actually see what happened.

This means setting up your application to record what’s happening in a way you can review afterward, both during development (to understand and iterate) and in production (to debug issues and monitor behavior). The ergonomics matter too: a plaintext dump of everything that happened isn’t a practical way to review agent behavior, even during development. You want tooling that lets you step through each decision and tool call interactively.

We recommend [Pydantic Logfire](https://logfire.pydantic.dev/docs/), which has been designed with Pydantic AI workflows in mind.

```
import logfire
logfire.configure()
logfire.instrument_pydantic_ai()
```
With Logfire instrumentation enabled, every agent run creates a detailed trace showing:

- **Messages exchanged**with the model (system, user, assistant)
- **Tool calls**including arguments and return values
- **Token usage**per request and cumulative
- **Latency**for each operation
- **Errors**with full context

This visibility is invaluable for:

- Understanding why an agent made a specific decision
- Debugging unexpected behavior
- Optimizing performance and costs
- Monitoring production deployments

For systematic evaluation of agent behavior beyond runtime debugging, [Pydantic Evals](/docs/ai/evals/evals) provides a code-first framework for testing AI systems:

```
from pydantic_evals import Case, Dataset
dataset = Dataset(
    name='agent_eval',
    cases=[
        Case(name='capital_question', inputs='What is the capital of France?', expected_output='Paris'),
    ]
)
report = dataset.evaluate_sync(my_agent_function)
```
Evals let you define test cases, run them against your agent, and score the results. When combined with Logfire, evaluation results appear in the web UI for visualization and comparison across runs. See the [Logfire integration guide](/docs/ai/evals/how-to/logfire-integration) for setup.

Pydantic AI’s instrumentation is built on [OpenTelemetry](https://opentelemetry.io/), so you can send traces to any compatible backend. Even if you use the Logfire SDK for its convenience, you can configure it to send data to other backends. See [alternative backends](/docs/ai/integrations/logfire#using-opentelemetry) for setup instructions.

[Full Logfire integration guide →](/docs/ai/integrations/logfire)

If models behave unexpectedly (e.g., the retry limit is exceeded, or their API returns `503`), agent runs will raise [ UnexpectedModelBehavior](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior).

In these cases, [ capture_run_messages](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) can be used to access the messages exchanged during the run to help diagnose the issue.

Define a tool that will raise `ModelRetry` repeatedly in this case.

[ capture_run_messages](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) is used to capture the messages exchanged during the run.

*(This example is complete, it can be run “as is”)*

When a run is cut short by an exception while streaming, an exception inside a tool, or external cancellation, Pydantic AI still captures partial state where it can. Partial [ ModelResponse](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) and 

[messages have](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)

`ModelRequest``state='interrupted'` so persistence layers and UIs can distinguish them from complete messages.For model responses, interrupted messages contain the response parts streamed before the interruption. For model requests, interrupted messages contain the tool results that completed before tool execution stopped. Half-finished tool call parts are not turned into synthetic tool results; only completed tool returns are captured.

In this example, `get_volume` completes before `get_mass` raises, so the interrupted request contains the completed `get_volume` return:

Agents can also be defined declaratively in YAML or JSON using [agent specs](/docs/ai/core-concepts/agent-spec). This separates agent configuration from application code:

```
model: anthropic:claude-opus-4-6
instructions: You are a helpful assistant.
capabilities:
  - WebSearch
  - Thinking:
      effort: high
```
```
from pydantic_ai import Agent
agent = Agent.from_file('agent.yaml')
```
See [Agent Specs](/docs/ai/core-concepts/agent-spec) for the full spec format, template strings, and custom capability registration.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/agent
