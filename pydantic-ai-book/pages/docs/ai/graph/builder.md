---
type: Web Page
title: Getting Started | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/builder
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Getting Started

The graph builder API provides a powerful builder pattern for constructing parallel execution graphs. The original `BaseNode`-based graph API is still available (and interoperable with the builder API) and is documented in the main graph documentation.

The graph builder API in `pydantic-graph` provides:

- **Step nodes**for executing async functions
- **Decision nodes**for conditional branching
- **Spread operations**for parallel processing of iterables
- **Broadcast operations**for sending the same data to multiple parallel paths
- **Join nodes and Reducers**for aggregating results from parallel execution

This API is designed for advanced workflows where you want declarative control over parallelism, routing, and data aggregation.

The graph builder API is included with `pydantic-graph`:

Or as part of `pydantic-ai`:

Here’s a simple example to get you started:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The `GraphBuilder` is the main entry point for constructing graphs. It’s generic over:

- `StateT`- The type of mutable state shared across all nodes
- `DepsT`- The type of dependencies injected into nodes
- `InputT`- The type of initial input to the graph
- `OutputT`- The type of final output from the graph

Steps are async functions decorated with `@g.step` that define the actual work to be done in each node. They receive a `StepContext` with access to:

- `ctx.state`- The mutable graph state
- `ctx.deps`- Injected dependencies
- `ctx.inputs`- Input data for this step

Edges define the connections between nodes. The builder provides multiple ways to create edges:

- `g.add()`- Add one or more edge paths
- `g.add_edge()`- Add a simple edge between two nodes
- `g.edge_from()`- Start building a complex edge path

Every graph has:

- `g.start_node`- The entry point receiving initial inputs
- `g.end_node`- The exit point producing final outputs

Here’s an example showcasing parallel execution with a map operation:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

In this example:

- The start node receives a list of integers
- The `.map()`operation fans out each item to a separate parallel execution of the`square`step
- All results are collected back together using `reduce_list_append`
- The joined results flow to the end node

Explore the detailed documentation for each feature:

- **Steps**- Learn about step nodes and execution contexts
- **Joins**- Understand join nodes and reducer patterns
- **Decisions**- Implement conditional branching
- **Parallel Execution**- Master broadcasting and mapping

Beyond the basic `graph.run()` method, the builder API provides fine-grained control over graph execution.

Use `graph.iter()` to execute the graph one step at a time:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

The `GraphRun` object provides:

- **Async iteration**: Iterate through execution events
- `next_task`property
- `output`property
- `next()`method

Generate Mermaid diagrams of your graph structure using `graph.render()`:

The rendered diagram can be displayed in documentation, notebooks, or any tool that supports Mermaid syntax.

The original graph API (documented in the main graph page) uses a class-based approach with `BaseNode` subclasses. The builder API uses a builder pattern with decorated functions, which provides:

**Advantages:**

- More concise syntax for simple workflows
- Explicit control over parallelism with map/broadcast
- Native reducers for common aggregation patterns
- Easier to visualize complex data flows

**Trade-offs:**

- Requires understanding of builder patterns
- Less object-oriented, more functional style

Both APIs are fully supported and can even be integrated together when needed.

For workflows that need to preserve progress across failures, restarts, or long-running operations, use one of the supported durable execution solutions.

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/builder
