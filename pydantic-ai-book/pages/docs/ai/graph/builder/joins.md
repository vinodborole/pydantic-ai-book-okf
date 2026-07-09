---
type: Web Page
title: Joins & Reducers | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/builder/joins
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Joins & Reducers

Join nodes synchronize and aggregate data from parallel execution paths. They use **Reducers** to combine multiple inputs into a single output.

When you use [parallel execution](/docs/ai/graph/builder/parallel) (broadcasting or mapping), you often need to collect and combine the results. Join nodes serve this purpose by:

- Waiting for all parallel tasks to complete
- Aggregating their outputs using a `ReducerFunction`
- Passing the aggregated result to the next node

Create a join using `GraphBuilder.join` with a reducer function and initial value or factory:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Pydantic Graph provides several common reducer types out of the box:

[ reduce_list_append](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.reduce_list_append) collects all inputs into a list:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

[ reduce_list_extend](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.reduce_list_extend) extends a list with an iterable of items:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

[ reduce_dict_update](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.reduce_dict_update) merges dictionaries together:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

[ reduce_null](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.reduce_null) discards all inputs and returns 

`None`. Useful when you only care about side effects:*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

[ reduce_sum](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.reduce_sum) sums numeric values:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

[ ReduceFirstValue](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.ReduceFirstValue) returns the first value it receives and cancels all other parallel tasks. This is useful for “race” scenarios where you want the first successful result:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Create custom reducers by defining a [ ReducerFunction](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.ReducerFunction):

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Reducers can access and modify the graph state:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Reducers with access to [ ReducerContext](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.ReducerContext) can call 

[to cancel all other parallel tasks in the same fork. This is useful for early termination when you’ve found what you need:](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join.ReducerContext.cancel_sibling_tasks)

`ctx.cancel_sibling_tasks()`*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Note that only 3 searches completed instead of all 5, because the reducer canceled the remaining tasks after finding a match.

A graph can have multiple independent joins:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Like steps, joins can have custom IDs:

Internally, the graph tracks which “fork” each parallel task belongs to. A join:

- Identifies its parent fork (the fork that created the parallel paths)
- Waits for all tasks from that fork to reach the join
- Calls `reduce()`for each incoming value
- Calls `finalize()`once all values are received
- Passes the finalized result to downstream nodes

This ensures proper synchronization even with nested parallel operations.

- Learn about [parallel execution](/docs/ai/graph/builder/parallel)with broadcasting and mapping
- Explore [conditional branching](/docs/ai/graph/builder/decisions)with decision nodes
- See the [API reference](/docs/ai/api/pydantic_graph/join/#pydantic_graph.join)for complete reducer documentation

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/builder/joins
