---
type: Web Page
title: Restate | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/durable_execution/restate
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Restate

Restate is a lightweight durable execution runtime with first-class support for AI agents. The Pydantic AI integration is provided via the Restate Python SDK.

Visit the Restate documentation for more information.

Restate makes your agent **durable** by recording every step of its execution in a journal. If your process crashes mid-execution, Restate replays the journal, skips completed steps, and resumes from exactly where it left off.

Your agent runs in a regular HTTP handler inside a Restate **service**. The Restate Server sits in front of your application and manages orchestration, journaling, and retries. Services run like regular Docker containers or serverless functions.

A durable agent has three building blocks:

- The **handler**: your agent logic, exposed as an HTTP endpoint in a Restate service.
- **LLM calls**: persisted so responses are not re-fetched on recovery — saving cost and time.
- **Tool executions**: wrapped in durable steps so side effects are not duplicated.

```
                  Clients
              (HTTP, Kafka, etc.)
                     |
                     v
            +---------------------+
            |   Restate Server    |      (Journals execution,
            +---------------------+       retries on failure,
                     ^                    manages state)
                     |
        Journal      |   Replay on
        steps,       |   recovery,
        retries      |   schedule calls
                     v
+------------------------------------------------------+
|               Application Process                    |
|   +----------------------------------------------+   |
|   |         Restate Service Handler              |   |
|   |           (Agent Run Loop)                   |   |
|   |    [ Durable Steps (Tool, MCP, Model) ]      |   |
|   +----------------------------------------------+   |
|         |           |                |               |
+------------------------------------------------------+
          |           |                |
          v           v                v
      [External APIs, services, databases, etc.]
```
See the Restate documentation for more information.

Any Pydantic AI agent can be made durable by wrapping it with `RestateAgent` from the Restate SDK and running it inside a Restate service handler.

Install the Restate SDK:

Here is a complete example of a durable Pydantic AI agent with Restate:

Define your agent and tools as you normally would with Pydantic AI.

Use `restate_context()` actions inside tools to make their execution durable. The result is persisted and retried until it succeeds. Side effects won't be duplicated on recovery.

`RestateAgent` wraps the agent so every LLM response is saved in the Restate Server and replayed during recovery.

The Restate service handler gives the agent a durable execution context and exposes it as an HTTP endpoint.

`restate.app()` creates the application that can be served.

Run the application with an ASGI server like Hypercorn.

See the Restate agent quickstart to learn how to run the agent.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/durable_execution/restate
