---
type: Web Page
title: Thread Executor | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/thread-executor
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Thread Executor

The `UseThreadExecutor`[capability](/docs/ai/capabilities/overview) provides a custom [`Executor`](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Executor) for running sync tool functions and other sync callbacks in threads. This is useful in long-running servers (e.g. FastAPI) where the default ephemeral threads from `anyio.to_thread.run_sync` can accumulate under sustained load:

```
from concurrent.futures import ThreadPoolExecutor
from pydantic_ai import Agent
from pydantic_ai.capabilities import UseThreadExecutor
executor = ThreadPoolExecutor(max_workers=16, thread_name_prefix='agent-worker')
agent = Agent('openai:gpt-5.2', capabilities=[UseThreadExecutor(executor)])
```
See [Thread executor for long-running servers](/docs/ai/tools-toolsets/tools-advanced#thread-executor-for-long-running-servers) for more details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/thread-executor
