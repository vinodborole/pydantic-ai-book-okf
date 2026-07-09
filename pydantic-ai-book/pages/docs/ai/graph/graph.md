---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/graph
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Overview

Graphs and finite state machines (FSMs) are a powerful abstraction to model, execute, control and visualize complex workflows.

Alongside Pydantic AI, we’ve developed `pydantic-graph` — an async graph and state machine library for Python where nodes and edges are defined using type hints.

While this library is developed as part of Pydantic AI; it has no dependency on `pydantic-ai` and can be considered as a pure graph-based state machine library. You may find it useful whether or not you’re using Pydantic AI or even building with GenAI.

`pydantic-graph` is designed for advanced users and makes heavy use of Python generics and type hints. It is not designed to be as beginner-friendly as Pydantic AI.

`pydantic-graph` is a required dependency of `pydantic-ai`, and an optional dependency of `pydantic-ai-slim`, see [installation instructions](/docs/ai/overview/install#slim-install) for more information. You can also install it directly:

`pydantic-graph` is made up of a few key components:

[ GraphRunContext](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.GraphRunContext) — The context for the graph run, similar to Pydantic AI’s 

[. This holds the state of the graph and dependencies and is passed to nodes when they’re run.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext``GraphRunContext` is generic in the state type of the graph it’s used in, [ StateT](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.StateT).

[ End](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End) — return value to indicate the graph run should end.

`End` is generic in the graph return type of the graph it’s used in, [ RunEndT](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.RunEndT).

Subclasses of [ BaseNode](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.BaseNode) define nodes for execution in the graph.

Nodes, which are generally [ dataclasses](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass), generally consist of:

- fields containing any parameters required/optional when calling the node
- the business logic to execute the node, in the `run`
- return annotations of the `run``pydantic-graph`to determine the outgoing edges of the node

Nodes are generic in:

- **state**, which must have the same type as the state of graphs they’re included in,- `StateT`- `None`, so if you’re not using state you can omit this generic parameter, see- [stateful graphs](#stateful-graphs)for more information
- **deps**, which must have the same type as the deps of the graph they’re included in,- `DepsT`- `None`, so if you’re not using deps you can omit this generic parameter, see- [dependency injection](#dependency-injection)for more information
- **graph return type**— this only applies if the node returns- `End`- `RunEndT`- [Never](https://docs.python.org/3/library/typing.html#typing.Never)so this generic parameter can be omitted if the node doesn’t return- `End`, but must be included if it does.

Here’s an example of a start or intermediate node in a graph — it can’t end the run as it doesn’t return [ End](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End):

State in this example is `MyState` (not shown), hence `BaseNode` is parameterized with `MyState`. This node can't end the run, so the `RunEndT` generic parameter is omitted and defaults to `Never`.

`MyNode` is a dataclass and has a single field `foo`, an `int`.

The `run` method takes a `GraphRunContext` parameter, again parameterized with state `MyState`.

The return type of the `run` method is `AnotherNode` (not shown), this is used to determine the outgoing edges of the node.

We could extend `MyNode` to optionally end the run if `foo` is divisible by 5:

We parameterize the node with the return type (`int` in this case) as well as state. Because generic parameters are positional-only, we have to include `None` as the second parameter representing deps.

The return type of the `run` method is now a union of `AnotherNode` and `End[int]`, this allows the node to end the run if `foo` is divisible by 5.

[ Graph](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph) — the executable graph produced by a 

[. The builder is the entry point for assembling a graph from](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder)

`GraphBuilder`[step functions](/docs/ai/graph/builder/steps),

[classes, and the edges connecting them.](#nodes)

`BaseNode`[ GraphBuilder](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder) is generic in:

- **state**the state type of the graph,- `StateT`
- **deps**the deps type of the graph,- `DepsT`
- **input**the type of the initial input passed to the graph,- `InputT`
- **output**the type of the final output produced by the graph,- `OutputT`

Here’s an example of a simple graph built from two `BaseNode` subclasses:

The `DivisibleBy5` node is parameterized with `None` for the state param and `None` for the deps param as this graph doesn't use state or deps, and `int` as it can end the run.

The `Increment` node doesn't return `End`, so the `RunEndT` generic parameter is omitted, state can also be omitted as the graph doesn't use state.

Create a [ GraphBuilder](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder) declaring the input and output types of the graph.

Define a [step](/docs/ai/graph/builder/steps) that wraps the initial input as the first `BaseNode`. The builder calls this when execution leaves [ g.start_node](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.start_node).

Register each `BaseNode` subclass with [ g.node()](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.node) so the builder knows about it; outgoing edges are inferred from each node's 

`run` return type.Wire the start node into the entry step.

[ graph.run()](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph.run) is async and returns the raw output value (the 

`int` returned by the `End` node).*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

A [mermaid diagram](#mermaid-diagrams) for this graph can be generated with `print(fives_graph)`, or by calling [ fives_graph.render()](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph.render):

```
stateDiagram-v2
  start
  DivisibleBy5
  state decision <<choice>>
  Increment
  [*] --> start
  start --> DivisibleBy5
  DivisibleBy5 --> decision
  decision --> Increment
  decision --> [*]
  Increment --> DivisibleBy5
```
The “state” concept in `pydantic-graph` provides an optional way to access and mutate an object (often a `dataclass` or Pydantic model) as nodes run in a graph. If you think of Graphs as a production line, then your state is the engine being passed along the line and built up by each node as the graph is run.

Here’s an example of a graph which represents a vending machine where the user may insert coins and select a product to purchase.

The state of the vending machine is defined as a dataclass with the user's balance and the product they've selected, if any.

A dictionary of products mapped to prices.

The `InsertCoin` node, [ BaseNode](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.BaseNode) is parameterized with 

`MachineState` as that's the state used in this graph.The `InsertCoin` node prompts the user to insert coins. We keep things simple by just entering a monetary amount as a float.

The `CoinsInserted` node; again this is a [ dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass) with one field 

`amount`.Update the user's balance with the amount inserted.

If the user has already selected a product, go to `Purchase`, otherwise go to `SelectProduct`.

In the `Purchase` node, look up the price of the product if the user entered a valid product.

If the user did enter a valid product, set the product in the state so we don't revisit `SelectProduct`.

If the balance is enough to purchase the product, adjust the balance to reflect the purchase and return [ End](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End) to end the graph. We're not using the run return type, so we call 

`End` with `None`.If the balance is insufficient, go to `InsertCoin` to prompt the user to insert more coins.

If the product is invalid, go to `SelectProduct` to prompt the user to select a product again.

Build the graph with [ GraphBuilder](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder), declaring the 

`MachineState` type. Each `BaseNode` subclass is registered with [; outgoing edges are inferred from the](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.node)

`g.node()``run` return types. The `start` step constructs the first node.The return type of the node's [ run](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.BaseNode.run) method is important as it is used to determine the outgoing edges of the node. This information in turn is used to render 

[mermaid diagrams](#mermaid-diagrams)and is enforced at runtime to detect misbehavior as soon as possible.

The return type of `CoinsInserted`'s [ run](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.BaseNode.run) method is a union, meaning multiple outgoing edges are possible.

Unlike other nodes, `Purchase` can end the run, so the [ RunEndT](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.RunEndT) generic parameter must be set. In this case it's 

`None` since the graph run return type is `None`.Initialize the state. This will be passed to the graph run and mutated as the graph runs.

Run the graph with the initial state. The first node to execute is determined by the `start` step we wired into [ g.start_node](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.start_node).

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

A [mermaid diagram](#mermaid-diagrams) for this graph can be generated with `print(vending_machine_graph)`:

```
stateDiagram-v2
  start
  InsertCoin
  CoinsInserted
  state decision <<choice>>
  Purchase
  SelectProduct
  state decision_2 <<choice>>
  [*] --> start
  start --> InsertCoin
  InsertCoin --> CoinsInserted
  CoinsInserted --> decision
  decision --> Purchase
  decision --> SelectProduct
  SelectProduct --> Purchase
  Purchase --> decision_2
  decision_2 --> InsertCoin
  decision_2 --> SelectProduct
  decision_2 --> [*]
```
See [below](#mermaid-diagrams) for more information on generating diagrams.

So far we haven’t shown an example of a Graph that actually uses Pydantic AI or GenAI at all.

In this example, one agent generates a welcome email to a user and the other agent provides feedback on the email.

This graph has a very simple structure:

```
---
title: feedback_graph
---
stateDiagram-v2
  [*] --> WriteEmail
  WriteEmail --> Feedback
  Feedback --> WriteEmail
  Feedback --> [*]
```
*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

For step-by-step execution — inspecting each task as it runs, overriding the next step, or driving the loop manually — use [ graph.iter()](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph.iter) instead of 

[. See](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph.run)

`graph.run()`[Advanced Execution Control](/docs/ai/graph/builder/#advanced-execution-control)in the graph builder docs for the iteration model and examples.

As with Pydantic AI, `pydantic-graph` supports dependency injection. Pass a `deps_type` to [ GraphBuilder](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder), parameterize each 

[subclass with the deps type, and read it via](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.BaseNode)

`BaseNode`[inside](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.GraphRunContext.deps)

`GraphRunContext.deps``run()` (or [inside step functions).](/docs/ai/api/pydantic_graph/step/#pydantic_graph.step.StepContext)

`StepContext.deps`As an example, let’s modify the `DivisibleBy5` example [above](#graph) to use a [ ProcessPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.ProcessPoolExecutor) to run the compute load in a separate process (this is a contrived example, 

`ProcessPoolExecutor` wouldn’t actually improve performance in this example):*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Pydantic Graph can render [mermaid](https://mermaid.js.org/) [ stateDiagram-v2](https://mermaid.js.org/syntax/stateDiagram.html) diagrams for any built graph. Call 

[(or just](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.Graph.render)

`graph.render()``print(graph)`) to get the mermaid source — pass `direction` (`'TB'`, `'LR'`, `'RL'`, or `'BT'`) to control layout. See the [graph builder mermaid section](/docs/ai/graph/builder/#mermaid-diagrams)for the full set of rendering options.

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/graph
