---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/durable_execution/overview
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Overview

Pydantic AI allows you to build durable agents that can preserve their progress across transient API failures and application errors or restarts, and handle long-running, asynchronous, and human-in-the-loop workflows with production-grade reliability. Durable agents have full support for [streaming](/docs/ai/core-concepts/agent#streaming-all-events) and [MCP](/docs/ai/mcp/client), with the added benefit of fault tolerance.

Pydantic AI officially supports four durable execution solutions:

These integrations are co-maintained by the Pydantic and vendor teams. The Temporal, DBOS, and Prefect integrations ship with Pydantic AI as [capabilities](/docs/ai/capabilities/overview) you attach to an agent; the [Restate](/docs/ai/capabilities/durable_execution/restate) integration lives in the Restate SDK and builds only on Pydantic AI’s public interface, so it can also serve as a reference for integrating with other durable systems.

Additional external SDK integrations:

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/durable_execution/overview
