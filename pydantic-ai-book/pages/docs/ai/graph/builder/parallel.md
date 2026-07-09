---
type: Web Page
title: Parallel Execution | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/builder/parallel
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Parallel Execution

The graph builder API provides two powerful mechanisms for parallel execution: **broadcasting** and **mapping**.

- **Broadcasting**- Send the same data to multiple parallel paths
- **Spreading**- Fan out items from an iterable to parallel paths

Both create “forks” in the execution graph that can later be synchronized with [join nodes](/docs/ai/graph/builder/joins).

Broadcasting sends identical data to multiple destinations simultaneously:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

All three steps receive the same input value (`10`) and execute in parallel.

Spreading fans out elements from an iterable, processing each element in parallel:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The `.map()` operation also works with `AsyncIterable` values. When mapping over an async iterable, the graph creates parallel tasks dynamically as values are yielded. This is particularly useful for streaming data or processing data that’s being generated on-the-fly:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

This allows for progressive processing where downstream steps can start working on early results while later results are still being generated.

The convenience method [ add_mapping_edge()](/docs/ai/api/pydantic_graph/graph_builder/#pydantic_graph.graph_builder.GraphBuilder.add_mapping_edge) provides a simpler syntax:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

When mapping an empty iterable, you can specify a `downstream_join_id` to ensure the join still executes:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

You can nest broadcasts and maps for complex parallel patterns:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The result contains:

- From 10: `10+1=11`and`10+2=12`
- From 20: `20+1=21`and`20+2=22`

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Add labels to parallel edges for better documentation:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

All parallel tasks share the same graph state. Be careful with mutations:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

You can transform data inline as it flows along edges using the `.transform()` method:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The transform function receives a [ StepContext](/docs/ai/api/pydantic_graph/step/#pydantic_graph.step.StepContext) with the current inputs and has access to state and dependencies. This is useful for:

- Converting data types between incompatible steps
- Extracting specific fields from complex objects
- Applying simple computations without creating a full step
- Adapting data formats during routing

Transforms can be chained and combined with other edge operations like `.map()` and `.label()`:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

- Learn about [join nodes](/docs/ai/graph/builder/joins)for aggregating parallel results
- Explore [conditional branching](/docs/ai/graph/builder/decisions)with decision nodes
- See the [steps documentation](/docs/ai/graph/builder/steps)for more on step execution

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/builder/parallel
