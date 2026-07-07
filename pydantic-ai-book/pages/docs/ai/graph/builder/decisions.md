---
type: Web Page
title: Decisions | Pydantic Docs
resource: https://pydantic.dev/docs/ai/graph/builder/decisions
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Decisions

Decision nodes enable conditional branching in your graph based on the type or value of data flowing through it.

A decision node evaluates incoming data and routes it to different branches based on:

- Type matching (using `isinstance`)
- Literal value matching
- Custom predicate functions

The first matching branch is taken, similar to pattern matching or `if-elif-else` chains.

Use `g.decision()` to create a decision node, then add branches with `g.match()`:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Match by type using regular Python types:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

For more complex type expressions like unions, you need to use `TypeExpression` because Python’s type system doesn’t allow union types to be used directly as runtime values:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Provide custom matching logic with the `matches` parameter:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Branches are evaluated in the order they’re added. The first matching branch is taken:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Both branches could match `10`, but Branch A is first, so it’s taken.

Use `object` or `Any` to create a catch-all branch:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Decisions can be nested for complex conditional logic:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

Add labels to branches for documentation and diagram generation:

*(This example is complete, it can be run “as is” — you’ll need to add  import asyncio; asyncio.run(main()) to run main)*

- Learn about parallel execution with broadcasting and mapping
- Understand join nodes for aggregating parallel results
- See the API reference for complete decision documentation

# Citations

1. Source page: https://pydantic.dev/docs/ai/graph/builder/decisions
