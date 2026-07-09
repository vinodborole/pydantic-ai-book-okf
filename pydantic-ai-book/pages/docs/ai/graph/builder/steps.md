---
type: Web Page
title: Steps | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/builder/steps
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Steps

Steps are the fundamental units of work in a graph. They’re async functions that receive a [ StepContext](/docs/ai/api/pydantic_graph/step/#pydantic_graph.step.StepContext) and return a value.

Steps are created using the [ @g.step](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.step) decorator on the 

[:](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder)

`GraphBuilder`*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Every step function receives a [ StepContext](/docs/ai/api/pydantic_graph/step/#pydantic_graph.step.StepContext) as its first parameter. The context provides access to:

- `ctx.state`- The mutable graph state (type:- `StateT`)
- `ctx.deps`- Injected dependencies (type:- `DepsT`)
- `ctx.inputs`- Input data for this step (type:- `InputT`)

State is shared across all steps in a graph and can be freely mutated:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Steps can receive and transform input data:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Steps can access injected dependencies through `ctx.deps`:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

By default, step node IDs are inferred from the function name. You can override this:

Labels provide documentation for diagram generation:

Multiple steps can be chained sequentially:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The computation is: `(10 + 5) * 2 - 3 = 27`

In addition to regular steps that return a single value, you can create streaming steps that yield multiple values over time using the [ @g.stream](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.stream) decorator:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Streaming steps return an `AsyncIterable` that yields values over time. When you use `.map()` on a streaming step’s output, the graph processes each yielded value as it becomes available, creating parallel tasks dynamically. This is particularly useful for:

- Processing data from APIs that stream responses
- Handling real-time data feeds
- Progressive processing of large datasets
- Any scenario where you want to start processing results before all data is available

Like regular steps, streaming steps can also have custom node IDs and labels:

The builder provides helper methods for common edge patterns:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The graph builder API provides strong type checking through generics. Type parameters on [ StepContext](/docs/ai/api/pydantic_graph/step/#pydantic_graph.step.StepContext) ensure:

- State access is properly typed
- Dependencies are correctly typed
- Input/output types match across edges

```
from dataclasses import dataclass
from pydantic_graph import GraphBuilder, StepContext
@dataclass
class MyState:
    pass
g = GraphBuilder(state_type=MyState, output_type=str)
# Type checker will catch mismatches
@g.step
async def expects_int(ctx: StepContext[MyState, None, int]) -> str:
    return str(ctx.inputs)
@g.step
async def returns_str(ctx: StepContext[MyState, None, None]) -> str:
    return 'hello'
# This would be a type error - expects_int needs int input, but returns_str outputs str
# g.add(g.edge_from(returns_str).to(expects_int))  # Type error!
```
- Learn about [parallel execution](/docs/ai/graph/builder/parallel)with broadcasting and mapping
- Understand [join nodes](/docs/ai/graph/builder/joins)for aggregating parallel results
- Explore [conditional branching](/docs/ai/graph/builder/decisions)with decision nodes

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/builder/steps
