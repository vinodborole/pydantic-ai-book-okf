---
type: Web Page
title: Agents | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/agent
timestamp: '2026-08-10T07:48:56.025339+00:00'
---

# Agents

Agents are Pydantic AI’s primary interface for interacting with LLMs.

In some use cases a single Agent will control an entire application or component, but multiple agents can also interact to embody more complex workflows.

The [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) class has full API documentation, but conceptually you can think of an agent as a container for:

| **Component** | **Description** | 
|---|---|
| [Instructions](#instructions) | A set of instructions for the LLM written by the developer. | 
| [Function tool(s)](/docs/ai/tools-toolsets/tools) and[toolsets](/docs/ai/tools-toolsets/toolsets) | Functions that the LLM may call to get information while generating a response. | 
| [Structured output type](/docs/ai/core-concepts/output) | The structured datatype the LLM must return at the end of a run, if specified. | 
| [Dependency type constraint](/docs/ai/core-concepts/dependencies) | Dynamic instructions functions, tools, and output functions may all use dependencies when they’re run. | 
| [LLM model](/docs/ai/api/models/base) | Optional default LLM model associated with the agent. Can also be specified when running the agent. | 
| [Model Settings](#additional-configuration) | Optional default model settings to help fine tune requests. Can also be specified when running the agent. | 
| [Capabilities](/docs/ai/capabilities/overview) | Reusable bundles of tools, hooks, instructions, and model settings that extend agent behavior. | 

While each of these can be configured individually, [capabilities](/docs/ai/capabilities/overview) let you bundle related behavior into reusable units that are easier to compose, share, and [load from configuration files](/docs/ai/core-concepts/agent-spec).

In typing terms, agents are generic in their dependency and output types, e.g., an agent which required dependencies of type `Foobar` and produced outputs of type `list[str]` would have type `Agent[Foobar, list[str]]`. In practice, you shouldn’t need to care about this, it should just mean your IDE can tell you when you have the right type, and if you choose to use [static type checking](#static-type-checking) it should work well with Pydantic AI.

Here’s a toy example of an agent that simulates a roulette wheel:

Create an agent, which expects an integer dependency and produces a boolean output. This agent will have type `Agent[int, bool]`.

Define a tool that checks if the square is a winner. Here [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is parameterized with the dependency type `int`; if you got the dependency type wrong you'd get a typing error.

In reality, you might want to use a random number here e.g. `random.randint(0, 36)`.

`result.output` will be a boolean indicating if the square is a winner. Pydantic performs the output validation, and it'll be typed as a `bool` since its type is derived from the `output_type` generic parameter of the agent.

There are five ways to run an agent:

1. [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) — an async function which returns a[`RunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) containing a completed response.
2. [`agent.run_sync()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync) — a plain, synchronous function which returns a[`RunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) containing a completed response (internally, this just calls`loop.run_until_complete(self.run())` ).
3. [`agent.run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) — an async context manager which returns a[`StreamedRunResult`](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult) , which contains methods to stream text and structured output as an async iterable.[`agent.run_stream_sync()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_sync) is a synchronous variation that returns a[`StreamedRunResultSync`](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResultSync) with synchronous versions of the same methods.
4. [`agent.run_stream_events()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events) — an async context manager which yields an async iterator over[`AgentStreamEvent`s](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent) ending with an[`AgentRunResultEvent`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResultEvent) containing the final run result.
5. [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) — a context manager which returns an[`AgentRun`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun) , an async iterable over the nodes of the agent’s underlying[`Graph`](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph) .

Here’s a simple example demonstrating the first four:

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

You can also pass messages from previous runs to continue a conversation or provide context, as described in [Messages and Chat History](/docs/ai/core-concepts/message-history).

As shown in the example above, [`run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) makes it easy to stream the agent’s final output as it comes in.
It also takes an optional `event_stream_handler` argument that you can use to gain insight into what is happening during the run before the final output is produced.

The example below shows how to stream events and text output. You can also [stream structured output](/docs/ai/core-concepts/output#streaming-structured-output).

These “dangling” tool calls will not be executed unless the agent’s [`end_strategy`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.end_strategy) is set to `'graceful'` or `'exhaustive'`, and even then their results will not be sent back to the model as the agent run will already be considered completed. In short, if the model returns both tool calls and text, and the agent’s output type is `str`, **the tool calls will not run** in streaming mode with the default setting.

If you want to always keep running the agent when it performs tool calls, and stream all events from the model’s streaming response and the agent’s execution of tools,
use [`agent.run_stream_events()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events) or [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.iter) instead, as described in the following sections.

*(This example is complete, it can be run “as is”)*

Like `agent.run_stream()`, [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) takes an optional `event_stream_handler`
argument that lets you stream all events from the model’s streaming response and the agent’s execution of tools.
Unlike `run_stream()`, it always runs the agent graph to completion even if text was received ahead of tool calls that looked like it could’ve been the final result.

For convenience, a [`agent.run_stream_events()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events) method is also available as a wrapper around `run(event_stream_handler=...)`. It is an async context manager that yields an async iterator over [`AgentStreamEvent`s](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent) ending with an [`AgentRunResultEvent`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResultEvent) carrying the final run result.

*(This example is complete, it can be run “as is”)*

Under the hood, each `Agent` in Pydantic AI uses **pydantic-graph** to manage its execution flow. **pydantic-graph** is a generic, type-centric library for building and running finite state machines in Python. It doesn’t actually depend on Pydantic AI — you can use it standalone for workflows that have nothing to do with GenAI — but Pydantic AI makes use of it to orchestrate the handling of model requests and model responses in an agent’s run.

In many scenarios, you don’t need to worry about pydantic-graph at all; calling `agent.run(...)` simply traverses the underlying graph from start to finish. However, if you need deeper insight or control — for example to inject your own logic at specific stages — Pydantic AI exposes the lower-level iteration process via [`Agent.iter`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter). This method returns an [`AgentRun`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun), which you can async-iterate over, or manually drive node-by-node via the [`next`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next) method. Once the agent’s graph returns an [`End`](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End), you have the final result along with a detailed history of all steps.

Here’s an example of using `async for` with `iter` to record each node the agent executes:

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

- The `AgentRun` is an async iterator that yields each node (`BaseNode` or`End` ) in the flow.
- The run ends when an `End` node is returned.

You can also drive the iteration manually by passing the node you want to run next to the `AgentRun.next(...)` method. This allows you to inspect or modify the node before it executes or skip nodes based on your own logic, and to catch errors in `next()` more easily:

We start by grabbing the first node that will be run in the agent's graph.

The agent run is finished once an `End` node has been produced; instances of `End` cannot be passed to `next`.

When you call `await agent_run.next(node)`, it executes that node in the agent's graph, updates the run's history, and returns the *next* node to run.

You could also inspect or mutate the new `node` here as needed.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

You can retrieve usage statistics (tokens, requests, etc.) at any time from the [`AgentRun`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun) object via `agent_run.usage`. This property returns a [`RunUsage`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage) object containing the usage data.

`RunUsage.cost` additionally holds a best-effort estimate of the run’s total cost in USD, calculated from each request’s usage with [genai-prices](https://github.com/pydantic/genai-prices). Requests to models or providers that genai-prices doesn’t have pricing data for don’t contribute to the total.

Once the run finishes, `agent_run.result` becomes an [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) object containing the final output (and related metadata).

Here is an example of streaming an agent run in combination with `async for` iteration:

*(This example is complete, it can be run “as is”)*

A run in flight can be cancelled entirely — e.g. when a user hits a “stop” button. Create a [`CancellationToken`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.CancellationToken), pass it to the run, and call `cancel()` from the stop handler. Cancellation raises [`RunCancelled`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled) with the completed message history and usage so you can persist and resume the conversation:

`cancel()` is idempotent and thread-safe. One token may govern multiple concurrent runs, cancelling all of them. A token is single-use: once cancelled it stays cancelled, and passing an already-cancelled token to a run prevents that run from starting (which also closes the "cancel raced ahead of the run" gap). So mint a fresh token per run or per stop gesture -- reusing one token across a session would cancel every run after the first before it starts.

[`RunCancelled.all_messages()`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled.all_messages) contains everything completed before cancellation, including completed tool results. Any dangling tool call is [repaired automatically](/docs/ai/core-concepts/message-history#making-histories-provider-valid) when the history is resumed.

[UI adapter](/docs/ai/integrations/ui/overview) users can persist this resumable history with the `on_cancel` callback.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

[`agent.run_sync()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync) accepts the same token. Calling `token.cancel()` from another thread is the only way to interrupt a synchronous run while it is blocked.

When the surrounding environment cancels the run — for example through `asyncio.timeout()`, a [`TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup), or application shutdown — the [`CancelledError`](https://docs.python.org/3/library/asyncio-exceptions.html#asyncio.CancelledError) remains unchanged. [`RunCancelled.from_cancellation()`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled.from_cancellation) provides the attached run state:

This demonstrates cancellation imposed by the surrounding asyncio environment. For application stop gestures, prefer a `CancellationToken`.

[`RunCancelled.all_messages()`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled.all_messages) contains everything completed before cancellation, including completed tool results. Any dangling tool call is [repaired automatically](/docs/ai/core-concepts/message-history#making-histories-provider-valid) when the history is resumed.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

On Python 3.10, asyncio recreates `CancelledError` across an `await task` boundary, but chains the original exception — carrying the attached run state — via `__context__`, which `from_cancellation()` traverses. The chain is attached only to the first `await` of the cancelled task, so later awaits of the same task see an unchained exception; [`capture_run_messages()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) is the fallback when only history is needed.

When consuming [`run_stream_events()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events), the yielded [`AgentRunEvents`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentRunEvents) handle offers a first-party alternative that needs no task juggling: [`AgentRunEvents.cancel()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentRunEvents.cancel) is safe to call from another task (e.g. a UI’s “stop” handler) and surfaces as `RunCancelled` on continued iteration:

Idempotent, a no-op once the run has finished, and callable before the first iteration to prevent the run from starting at all.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

Externally cancelling the consuming task works here too: the background run tears down, the propagating `CancelledError` carries the run state for `from_cancellation()`, and the handle’s `all_messages()` and `usage` remain accessible afterwards.

To request cancellation from a tool, an `event_stream_handler`, or a capability hook, call [`RunContext.cancel()`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.cancel). This requests first-party cancellation, so the run ends with [`RunCancelled`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled) rather than an external `CancelledError`. `cancel()` itself returns normally — the cancellation is delivered at the calling code’s next `await`, and the tool’s return value is discarded — so a tool can still run cleanup after requesting it:

You may not control which way cancellation will arrive: a caller wraps `agent.run()` in a task for a stop gesture, while a tool — perhaps from another library — calls `ctx.cancel()` internally. Handle each on its own terms — consume the first-party `RunCancelled`, but let an external `CancelledError` keep propagating so timeouts and task groups still tear down correctly, capturing its state first if you need it:

Here the tool cancels first-party, so `await task` raises `RunCancelled`. Had a stop button called `task.cancel()` instead, `await task` would raise `CancelledError` and the second handler would run.

First-party cancellation is a `RunCancelled` you can consume: the run stopped because your own code asked it to, so returning normally is fine.

External cancellation stays `CancelledError`, and a stop button's `task.cancel()` is indistinguishable from a timeout or a [`TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup) tearing down -- so re-raise it (swallowing it would break those teardowns), reaching for [`from_cancellation()`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled.from_cancellation) only to capture the partial state first. It returns `None` when nothing is attached, e.g. an application shutdown unrelated to this run.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

Cancellation is terminal: capability hooks may observe it and clean up, but cannot recover the run to success — on Python 3.11+ this holds even if user code absorbs the delivered cancellation; on Python 3.10 it is best-effort. When first-party and external cancellation race, external cancellation wins. On Python 3.10, that race cannot be distinguished, so first-party cancellation wins instead.

For fine-grained control over the agent graph, call [`AgentRun.cancel()`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.cancel) on the handle returned by [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter):

`AgentRun.cancel()` is safe to call from another task and is a no-op once the run has finished.

Inside the `agent.iter()` block, cancellation surfaces as `asyncio.CancelledError`; after the context exits, first-party cancellation raises `RunCancelled` with a detached state snapshot.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

When a stream is cancelled mid-generation, the response is recorded with `state='interrupted'` in the message history. The history includes any partial content that was received before cancellation:

The message history includes the interrupted response with any partial content that was received before cancellation.

The interrupted response state lets your application decide whether to keep, inspect, or discard the partial response before reusing the history.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

Cancellation is **run-scoped**: `cancel()` cancels the run its `RunContext` belongs to, and a `CancellationToken` cancels the runs it’s attached to. This matters when you use [agent delegation](/docs/ai/guides/multi-agent-applications#agent-delegation) — a tool that runs another agent with `await sub_agent.run(...)`:

- **A sub-agent cancelling itself does not cancel the parent** — when it’s`await` ed inside a tool body. If the sub-agent (or one of its tools) calls`ctx.cancel()` , that cancels the*sub-agent’s* run. The delegate tool sees a[`RunCancelled`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled) , which — if it isn’t caught — surfaces to the parent as a*failed tool return* the parent’s model can react to, not as a cancellation of the parent run. This isolation is specific to tool bodies: a sub-agent`await` ed from an`event_stream_handler` , an[output validator](/docs/ai/core-concepts/output#output-validators) , or a[capability](/docs/ai/capabilities/overview) hook runs directly on the parent’s task, so its`cancel()`*does* surface as the parent’s own`RunCancelled` .
- **To cancel the parent too, opt in from the delegate tool** by catching`RunCancelled` and calling`ctx.cancel()` on the parent’s context (or re-raising a different error).
- **To cancel a whole tree of runs at once, share one `CancellationToken`** across the parent and its sub-agents — cancelling it stops all of them. A parent cancelled this way (or by an external`asyncio.CancelledError` ) also tears down any sub-agent run it is`await` ing inline, since they run on the same task.

Pydantic AI offers a [`UsageLimits`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) structure to help you limit your
usage (tokens, requests, tool calls, and cost) on model runs.

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
#> RunUsage(cost=Decimal('0.000201'), input_tokens=62, output_tokens=1, requests=1)
try:
    result_sync = agent.run_sync(
        'What is the capital of Italy? Answer with a paragraph.',
        usage_limits=UsageLimits(output_tokens_limit=10),
    )
except UsageLimitExceeded as e:
    print(e)
    """
    Exceeded the output_tokens_limit of 10 (output_tokens=32). Consider raising the limit, or see the docs on usage limits for budget-aware patterns: https://ai.pydantic.dev/agent/#usage-limits
    """
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
    """
    The next tool call(s) would exceed the tool_calls_limit of 1 (tool_calls=2). Consider raising the limit, or see the docs on usage limits for budget-aware patterns: https://ai.pydantic.dev/agent/#usage-limits
    """
```
Tools and [capabilities](/docs/ai/capabilities/overview) can read the run’s limits from [`ctx.usage_limits`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.usage_limits) (alongside [`ctx.usage`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.usage) for usage so far), so a budget-aware tool or capability can disclose or adapt to the remaining budget without being configured with a duplicate copy of the limits. It reflects what the run is already enforcing and is read-only by convention.

The token limits above are cumulative across the whole run. To instead cap the size of any single request’s input (the context window actually sent to the model), use `per_request_input_tokens_limit`. This is useful when prompt caching makes cumulative input a poor proxy for cost: re-sent cached prefixes are cheap, while a single oversized context is what degrades model performance and drives cache-miss cost.

```
from pydantic_ai import Agent, UsageLimitExceeded, UsageLimits
agent = Agent('anthropic:claude-sonnet-4-6')
try:
    agent.run_sync(
        'What is the capital of Italy? Answer with just the city.',
        usage_limits=UsageLimits(per_request_input_tokens_limit=10),
    )
except UsageLimitExceeded as e:
    print(e)
    """
    Exceeded the per_request_input_tokens_limit of 10 (request_input_tokens=62). Consider raising the limit, or see the docs on usage limits for budget-aware patterns: https://ai.pydantic.dev/agent/#usage-limits
    """
```
By default the limit is checked against the provider-reported input tokens after the response, so the oversized request is still sent and billed (matching `input_tokens_limit`). Set `count_tokens_before_request=True` to run a token-counting pass and enforce the limit before the request is sent.

Token limits are a proxy for spend: the same token count costs wildly different amounts on different models, so a limit tuned for one model is wrong for the next. To bound the actual dollars a run can spend, use [`cost_limit`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits.cost_limit), which caps `RunUsage.cost` in USD:

```
from decimal import Decimal
from pydantic_ai import Agent, UsageLimitExceeded, UsageLimits
agent = Agent('anthropic:claude-sonnet-4-6')
try:
    agent.run_sync(
        'What is the capital of Italy? Answer with just the city.',
        usage_limits=UsageLimits(cost_limit=Decimal('0.0001')),
    )
except UsageLimitExceeded as e:
    print(e)
    """
    Exceeded the `cost_limit` of 0.0001 (`usage.cost`=Decimal('0.000201')). Consider raising the limit, or see the docs on usage limits for budget-aware patterns: https://ai.pydantic.dev/agent/#usage-limits
    """
```
Like `output_tokens_limit`, this is checked after each response, since a response’s output cost isn’t known until it arrives. Setting `count_tokens_before_request=True` additionally prices the counted input tokens and rejects the request up front when that lower bound alone exceeds the limit.

Pydantic AI offers a [`settings.ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) structure to help you fine tune your requests.
This structure allows you to configure common parameters that influence the model’s behavior, such as `temperature`, `max_tokens`, `top_k`,
`timeout`, and more.

There are three ways to apply these settings, with a clear precedence order:

1. **Model-level defaults** - Set when creating a model instance via the`settings` parameter. These serve as the base defaults for that model.
2. **Agent-level defaults** - Set during[`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) initialization via the`model_settings` argument. These are merged with model defaults, with agent settings taking precedence.
3. **Run-time overrides** - Passed to`run{_sync,_stream}` functions via the`model_settings` argument. These have the highest priority and are merged with the combined agent and model defaults.

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
[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings).
The callable is invoked before each model request, so settings can vary per step.
The current resolved settings so far are available via `ctx.model_settings` inside the callable.

Settings are resolved in layers, each merged on top of the previous:

1. **Model defaults** (`model.settings` )
2. **Agent-level** (`Agent(model_settings=...)` )
3. **Capability-level** (e.g. from[`Thinking()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Thinking) — see[Capabilities](/docs/ai/capabilities/custom#providing-model-settings) )
4. **Run-level** (`agent.run(model_settings=...)` )

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
and read it after completion via [`AgentRun.metadata`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun),
[`AgentRunResult.metadata`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult), or
[`StreamedRunResult.metadata`](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult).
The resolved metadata is attached to the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) during the run and,
when instrumentation is enabled, added to the run span attributes for observability tools.

Configure metadata on an [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) or pass it to a run.
Both accept either a static dictionary or a callable that receives the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext).
Metadata is computed (if a callable) and applied when the run starts, then recomputed after a run ends successfully,
so it can include end-of-run values.
Agent-level metadata and per-run metadata are merged, with per-run values overriding agent-level ones.

You can limit the number of concurrent agent runs using the `max_concurrency` parameter.
This is useful when you want to prevent overwhelming external resources or enforce rate limits when running many agent instances in parallel.

When the concurrency limit is reached, additional calls to [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) or [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter)
will wait until a slot becomes available. If you configure `max_queued` and the queue fills up,
a [`ConcurrencyLimitExceeded`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ConcurrencyLimitExceeded) exception is raised.

When instrumentation is enabled, waiting operations appear as “waiting for concurrency” spans with attributes showing queue depth and limits.

If you wish to further customize model behavior, you can use a subclass of [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings), like
[`GoogleModelSettings`](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModelSettings), associated with your model of choice.

For example:

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

1. **Static system prompts** : These are known when writing the code and can be defined via the`system_prompt` parameter of the[`Agent` constructor](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) .
2. **Dynamic system prompts** : These depend in some way on context that isn’t known until runtime, and should be defined via functions decorated with[`@agent.system_prompt`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.system_prompt) .

You can add both to a single agent; they’re appended in the order they’re defined at runtime.

Here’s an example using both types of system prompts:

The agent expects a string dependency.

Static system prompt defined at agent creation time.

Dynamic system prompt defined via a decorator with [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), this is called just after `run_sync`, not when the agent is created, so can benefit from runtime information like the dependencies used on that run.

Another dynamic system prompt, system prompts don't have to have the `RunContext` parameter.

*(This example is complete, it can be run “as is”)*

Instructions are similar to system prompts. The main difference is that when an explicit `message_history` is provided
in a call to `Agent.run` and similar methods, *instructions* from any existing messages in the history are not included
in the request to the model — only the instructions of the *current* agent are included.

You should use:

- `instructions` when you want your request to the model to only include system prompts for the*current* agent
- `system_prompt` when you want your request to the model to*retain* the system prompts used in previous requests (possibly made using other agents)

In general, we recommend using `instructions` instead of `system_prompt` unless you have a specific reason to use `system_prompt`.

Instructions, like system prompts, can be specified at different times:

1. **Static instructions** : These are known when writing the code and can be defined via the`instructions` parameter of the[`Agent` constructor](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) .
2. **Dynamic instructions** : These rely on context that is only available at runtime and should be defined using functions decorated with[`@agent.instructions`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.instructions) . Unlike dynamic system prompts, which may be reused when`message_history` is present, dynamic instructions are always reevaluated.
3. **Runtime instructions** : These are additional instructions for a specific run that can be passed to one of the[run methods](#running-agents) using the`instructions` argument.

All three types of instructions can be added to a single agent, and they are appended in the order they are defined at runtime. Each instruction is internally classified as either **static** (literal strings from the `instructions` parameter) or **dynamic** (from `@agent.instructions` functions, runtime instructions, or [toolset](/docs/ai/tools-toolsets/toolsets) instructions). Static instructions are always sorted before dynamic ones. This ordering enables providers that support prompt caching (like [Anthropic](/docs/ai/models/anthropic#smart-instruction-caching) and [Bedrock](/docs/ai/models/bedrock#prompt-caching)) to cache the stable static prefix while leaving dynamic instructions outside the cache boundary.

Here’s an example using a static instruction as well as dynamic instructions:

The agent expects a string dependency.

Static instructions defined at agent creation time.

Dynamic instructions defined via a decorator with [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext),
this is called just after `run_sync`, not when the agent is created, so can benefit from runtime
information like the dependencies used on that run.

Another dynamic instruction, instructions don't have to have the `RunContext` parameter.

*(This example is complete, it can be run “as is”)*

Note that returning an empty string will result in no instruction message added.

Instructions can also come from [capabilities](/docs/ai/capabilities/overview) via [`get_instructions()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions), or from [template strings](/docs/ai/core-concepts/agent-spec#template-strings) rendered against the agent’s dependencies.

Validation errors from both function tool parameter validation and [structured output validation](/docs/ai/core-concepts/output#structured-output) can be passed back to the model with a request to retry.

You can also raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) from within a [tool](/docs/ai/tools-toolsets/tools) or [output function](/docs/ai/core-concepts/output#output-functions) to tell the model it should retry generating a response.

- The default retry count is **1** but can be altered for the[entire agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) with`retries` or[`AgentRetries`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentRetries) , a[specific tool](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) , or[outputs](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) . Both the tool and output sides of the agent retry budget can also be overridden per run via`agent.run(retries={'tools': ..., 'output': ...})` and friends (or for a block of runs via[`agent.override()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) ). At these call sites a bare`int` overrides both budgets, just like at construction — pass a dict such as`retries={'tools': ...}` to override just one. The tool-retry default and its per-run override apply to function tools, output tools, and MCP tools.
- You can access the current retry count from within a tool, output validator, or output function via [`ctx.retry`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.retry) .

Pydantic AI enforces the output retry budget differently depending on how the model returns its final output:

- **Text output path** (`output_type=str` , text-only outputs, empty or unusable model responses): a single global budget is shared across the whole run. Each invalid response consumes one unit of the budget; when it’s exhausted, the run raises[`UnexpectedModelBehavior`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior) with message`'Exceeded maximum output retries (N)'` .
- **Tool output path** ([`output_type=ToolOutput(...)`](/docs/ai/core-concepts/output#tool-output) , structured outputs): the output retry budget is the*default per-tool limit* . See[Tool Output](/docs/ai/core-concepts/output#tool-output) for per-tool overrides via[`ToolOutput(max_retries=N)`](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput.max_retries) .

For how the budget appears inside [output validators](/docs/ai/core-concepts/output#output-validator-functions) — including what `ctx.max_retries` and `ctx.retry` reflect on each path — see the [Output validators](/docs/ai/core-concepts/output#output-validator-functions) section.

Tool retries are tracked per tool — see [Tool Execution, Retries, and Failures](/docs/ai/tools-toolsets/tools-advanced#tool-retries) for the per-tool counter model and the three configuration levels.

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

- **Messages exchanged** with the model (system, user, assistant)
- **Tool calls** including arguments and return values
- **Token usage** per request and cumulative
- **Latency** for each operation
- **Errors** with full context

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

If models behave unexpectedly (e.g., the retry limit is exceeded, or their API returns `503`), agent runs will raise [`UnexpectedModelBehavior`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior).

In these cases, [`capture_run_messages`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) can be used to access the messages exchanged during the run to help diagnose the issue.

For a run that was cancelled rather than failed, [`RunCancelled`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled) and [`RunCancelled.from_cancellation()`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.RunCancelled.from_cancellation) carry the run’s history directly — see [Cancelling a Run](#cancelling-a-run).

Define a tool that will raise `ModelRetry` repeatedly in this case.

[`capture_run_messages`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) is used to capture the messages exchanged during the run.

*(This example is complete, it can be run “as is”)*

When a run is cut short by an exception while streaming, an exception inside a tool, or external cancellation, Pydantic AI still captures partial state where it can. Partial [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) and [`ModelRequest`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest) messages have `state='interrupted'` so persistence layers and UIs can distinguish them from complete messages.

For model responses, interrupted messages contain the response parts streamed before the interruption. For model requests, interrupted messages contain the tool results that completed before tool execution stopped. The captured messages reflect exactly what happened — half-finished tool call parts are not turned into synthetic tool results at capture time. When an interrupted history is passed back into a run, it is [repaired automatically](/docs/ai/core-concepts/message-history#making-histories-provider-valid) before the next model request.

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
