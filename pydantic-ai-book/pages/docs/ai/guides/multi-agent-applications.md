---
type: Web Page
title: Multi-Agent Patterns | Pydantic Docs
resource: https://pydantic.dev/docs/ai/guides/multi-agent-applications
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Multi-Agent Patterns

There are roughly five levels of complexity when building applications with Pydantic AI:

- Single agent workflows — what most of the `pydantic_ai`documentation covers
- [Agent delegation](#agent-delegation)— agents using another agent via tools
- [Programmatic agent hand-off](#programmatic-agent-hand-off)— one agent runs, then application code calls another agent
- [Graph based control flow](/docs/ai/graph/graph)— for the most complex cases, a graph-based state machine can be used to control the execution of multiple agents
- [Deep Agents](#deep-agents)— autonomous agents with planning, file operations, task delegation, and sandboxed code execution

Of course, you can combine multiple strategies in a single application.

“Agent delegation” refers to the scenario where an agent delegates work to another agent, then takes back control when the delegate agent (the agent called from within a tool) finishes.
If you want to hand off control to another agent completely, without coming back to the first agent, you can use an [output function](/docs/ai/core-concepts/output#output-functions).

Since agents are stateless and designed to be global, you do not need to include the agent itself in agent [dependencies](/docs/ai/core-concepts/dependencies).

You’ll generally want to pass [ ctx.usage](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.usage) to the 

[keyword argument of the delegate agent run so usage within that run counts towards the total usage of the parent agent run.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run)

`usage`The "parent" or controlling agent.

Passing `name` is optional but recommended when you run more than one agent: it labels each agent's run span, so naming both lets you tell the parent and delegate apart in [Logfire](/docs/ai/integrations/logfire). When omitted, the name is inferred from the variable the agent is assigned to and falls back to `'agent'` when it can't be (e.g. agents kept in a list or dict).

The "delegate" agent, which is called from within a tool of the parent agent.

Call the delegate agent from within a tool of the parent agent.

Pass the usage from the parent agent to the delegate agent so the final [ result.usage](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.usage) includes the usage from both agents.

Since the function returns `list[str]`, and the `output_type` of `joke_generation_agent` is also `list[str]`, we can simply return `r.output` from the tool.

*(This example is complete, it can be run “as is”)*

The control flow for this example is pretty simple and can be summarised as follows:

```
graph TD
  START --> joke_selection_agent
  joke_selection_agent --> joke_factory["joke_factory (tool)"]
  joke_factory --> joke_generation_agent
  joke_generation_agent --> joke_factory
  joke_factory --> joke_selection_agent
  joke_selection_agent --> END
```
Generally the delegate agent needs to either have the same [dependencies](/docs/ai/core-concepts/dependencies) as the calling agent, or dependencies which are a subset of the calling agent’s dependencies.

Define a dataclass to hold the client and API key dependencies.

Set the `deps_type` of the calling agent — `joke_selection_agent` here.

Pass the dependencies to the delegate agent's run method within the tool call.

Also set the `deps_type` of the delegate agent — `joke_generation_agent` here.

Define a tool on the delegate agent that uses the dependencies to make an HTTP request.

Usage now includes 4 requests — 2 from the calling agent and 2 from the delegate agent.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

This example shows how even a fairly simple agent delegation can lead to a complex control flow:

```
graph TD
  START --> joke_selection_agent
  joke_selection_agent --> joke_factory["joke_factory (tool)"]
  joke_factory --> joke_generation_agent
  joke_generation_agent --> get_jokes["get_jokes (tool)"]
  get_jokes --> http_request["HTTP request"]
  http_request --> get_jokes
  get_jokes --> joke_generation_agent
  joke_generation_agent --> joke_factory
  joke_factory --> joke_selection_agent
  joke_selection_agent --> END
```
“Programmatic agent hand-off” refers to the scenario where multiple agents are called in succession, with application code and/or a human in the loop responsible for deciding which agent to call next.

Here agents don’t need to use the same deps.

Here we show two agents used in succession, the first to find a flight and the second to extract the user’s seat preference.

Define the first agent, which finds a flight. We use an explicit type annotation until [PEP-747](https://peps.python.org/pep-0747/) lands, see [structured output](/docs/ai/core-concepts/output#structured-output). We use a union as the output type so the model can communicate if it's unable to find a satisfactory choice; internally, each member of the union will be registered as a separate tool.

Define a tool on the agent to find a flight. In this simple case we could dispense with the tool and just define the agent to return structured data, then search for a flight, but in more complex scenarios the tool would be necessary.

Define usage limits for the entire app.

Define a function to find a flight, which asks the user for their preferences and then calls the agent to find a flight.

As with `flight_search_agent` above, we use an explicit type annotation to define the agent.

Define a function to find the user's seat preference, which asks the user for their seat preference and then calls the agent to extract the seat preference.

Now that we've put our logic for running each agent into separate functions, our main app becomes very simple.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The control flow for this example can be summarised as follows:

```
graph TB
  START --> ask_user_flight["ask user for flight"]
  subgraph find_flight
    flight_search_agent --> ask_user_flight
    ask_user_flight --> flight_search_agent
  end
  flight_search_agent --> ask_user_seat["ask user for seat"]
  flight_search_agent --> END
  subgraph find_seat
    seat_preference_agent --> ask_user_seat
    ask_user_seat --> seat_preference_agent
  end
  seat_preference_agent --> END
```
See the [graph](/docs/ai/graph/graph) documentation on when and how to use graphs.

Deep agents are autonomous agents that combine multiple architectural patterns and capabilities to handle complex, multi-step tasks reliably. These patterns can be implemented using Pydantic AI’s built-in features and (third-party) toolsets:

- **Planning and progress tracking**— agents break down complex tasks into steps and track their progress, giving users visibility into what the agent is working on. See- [Task Management toolsets](/docs/ai/tools-toolsets/toolsets#task-management).
- **File system operations**— reading, writing, and editing files with proper abstraction layers that work across in-memory storage, real file systems, and sandboxed containers. See- [File Operations toolsets](/docs/ai/tools-toolsets/toolsets#file-operations).
- **Task delegation**— spawning specialized sub-agents for specific tasks, with isolated context to prevent recursive delegation issues. See- [Agent Delegation](#agent-delegation)above.
- **Sandboxed code execution**— running AI-generated code in isolated environments (typically Docker containers) to prevent accidents. See- [Code Execution toolsets](/docs/ai/tools-toolsets/toolsets#code-execution).
- **Context management**— automatic conversation summarization to handle long sessions that would otherwise exceed token limits. See- [Processing Message History](/docs/ai/core-concepts/message-history#processing-message-history).
- **Human-in-the-loop**— approval workflows for dangerous operations like code execution or file deletion. See- [Requiring Tool Approval](/docs/ai/tools-toolsets/toolsets#requiring-tool-approval).
- **Durable execution**— preserving agent state across transient API failures and application errors or restarts. See- [Durable Execution](/docs/ai/integrations/durable_execution/overview).

In addition, the community maintains packages that bring these concepts together in a more opinionated way:

Multi-agent systems can be challenging to debug due to their complexity; when multiple agents interact, understanding the flow of execution becomes essential.

With [Logfire](/docs/ai/integrations/logfire), you can trace the entire flow across multiple agents:

```
import logfire
logfire.configure()
logfire.instrument_pydantic_ai()
# Your multi-agent code here...
```
Logfire shows you:

- **Which agent handled which part**of the request
- **Delegation decisions**—when and why one agent called another
- **End-to-end latency**broken down by agent
- **Token usage and costs**per agent
- **What triggered the agent run**—the HTTP request, scheduled job, or user action that started it all
- **What happened inside tool calls**—database queries, HTTP requests, file operations, and any other instrumented code that tools execute

This is essential for understanding and optimizing complex agent workflows. When something goes wrong in a multi-agent system, you’ll see exactly which agent failed and what it was trying to do, and whether the problem was in the agent’s reasoning or in the backend systems it called.

If your Pydantic AI application includes a TypeScript frontend, API gateway, or services in other languages, Logfire can trace them too—Logfire provides SDKs for Python, JavaScript/TypeScript, and Rust, plus compatibility with any OpenTelemetry-instrumented application. See traces from your entire stack in a unified view. For details on sending data from other languages using standard OpenTelemetry, see the [alternative clients guide](https://logfire.pydantic.dev/docs/how-to-guides/alternative-clients/).

Pydantic AI’s instrumentation is built on [OpenTelemetry](https://opentelemetry.io/), so you can also use any OTel-compatible backend. See the [Logfire integration guide](/docs/ai/integrations/logfire) for details.

The following examples demonstrate how to use multi-agent patterns in Pydantic AI:

# Citations

1. Source page: https://pydantic.dev/docs/ai/guides/multi-agent-applications
