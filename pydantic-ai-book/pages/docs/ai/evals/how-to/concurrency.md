---
type: Web Page
title: Concurrency & Performance | Pydantic Docs
resource: https://pydantic.dev/docs/ai/evals/how-to/concurrency
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Concurrency & Performance

Control how evaluation cases are executed in parallel.

By default, Pydantic Evals runs all cases concurrently to maximize throughput. You can control this behavior using the `max_concurrency` parameter.

```
from pydantic_evals import Case, Dataset
def my_task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(name='concurrency_demo', cases=[Case(inputs='test1'), Case(inputs='test2')])
# Run all cases concurrently (default)
report = dataset.evaluate_sync(my_task)
# Limit to 5 concurrent cases
report = dataset.evaluate_sync(my_task, max_concurrency=5)
# Run sequentially (one at a time)
report = dataset.evaluate_sync(my_task, max_concurrency=1)
```
Many APIs have rate limits that restrict concurrent requests:

```
from pydantic_evals import Case, Dataset
async def my_llm_task(inputs: str) -> str:
    return f'LLM Result: {inputs}'
dataset = Dataset(name='rate_limit_demo', cases=[Case(inputs='test1')])
# If your API allows 10 requests/second
report = dataset.evaluate_sync(
    my_llm_task,
    max_concurrency=10,
)
```
Limit concurrency to avoid overwhelming system resources:

```
from pydantic_evals import Case, Dataset
def heavy_computation(inputs: str) -> str:
    return f'Heavy: {inputs}'
def db_query_task(inputs: str) -> str:
    return f'DB: {inputs}'
dataset = Dataset(name='resource_constraints', cases=[Case(inputs='test1')])
# Memory-intensive operations
report = dataset.evaluate_sync(
    heavy_computation,
    max_concurrency=2,  # Only 2 at a time
)
# Database connection pool limits
report = dataset.evaluate_sync(
    db_query_task,
    max_concurrency=5,  # Match connection pool size
)
```
Run sequentially to see clear error traces:

```
from pydantic_evals import Case, Dataset
def my_task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(name='debug_demo', cases=[Case(inputs='test1')])
# Easier to debug
report = dataset.evaluate_sync(
    my_task,
    max_concurrency=1,
)
```
Here’s an example showing the performance difference:

Both task execution and evaluator execution happen concurrently by default:

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import LLMJudge
def my_task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(
    name='evaluator_concurrency',
    cases=[Case(inputs=f'test{i}') for i in range(100)],  # 100 cases
    evaluators=[
        LLMJudge(rubric='Quality check'),  # Makes API calls
    ],
)
# Both task and evaluator run with controlled concurrency
report = dataset.evaluate_sync(
    my_task,
    max_concurrency=10,
)
```
If your evaluators are expensive (e.g., [ LLMJudge](/docs/ai/api/pydantic_evals/evaluators/#pydantic_evals.evaluators.LLMJudge)), limiting concurrency helps manage:

- API rate limits
- Cost (fewer concurrent API calls)
- Memory usage

Both sync and async evaluation support concurrency control:

```
from pydantic_evals import Case, Dataset
def my_task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(name='sync_demo', cases=[Case(inputs='test1')])
# Runs async operations internally with controlled concurrency
report = dataset.evaluate_sync(my_task, max_concurrency=10)
```
```
from pydantic_evals import Case, Dataset
async def my_task(inputs: str) -> str:
    return f'Result: {inputs}'
async def run_evaluation():
    dataset = Dataset(name='async_demo', cases=[Case(inputs='test1')])
    # Same behavior, but in async context
    report = await dataset.evaluate(my_task, max_concurrency=10)
    return report
```
Track execution to optimize settings:

```
import time
from pydantic_evals import Case, Dataset
def task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(name='monitoring', cases=[Case(inputs=f'test{i}') for i in range(10)])
t0 = time.time()
report = dataset.evaluate_sync(task, max_concurrency=10)
duration = time.time() - t0
num_cases = len(report.cases) + len(report.failures)
avg_duration = duration / num_cases
print(f'Total: {duration:.2f}s')
#> Total: 0.01s
print(f'Cases: {num_cases}')
#> Cases: 10
print(f'Avg per case: {avg_duration:.2f}s')
#> Avg per case: 0.00s
print(f'Effective concurrency: ~{num_cases * avg_duration / duration:.1f}')
#> Effective concurrency: ~1.0
```
If you hit rate limits, the evaluation will fail. Use retry strategies:

```
from pydantic_evals import Case, Dataset
def task(inputs: str) -> str:
    return f'Result: {inputs}'
dataset = Dataset(name='rate_limit_handling', cases=[Case(inputs='test1')])
# Reduce concurrency to avoid rate limits
report = dataset.evaluate_sync(
    task,
    max_concurrency=5,  # Stay under rate limit
)
```
See [Retry Strategies](/docs/ai/evals/how-to/retry-strategies) for handling transient failures.

- [Retry Strategies](/docs/ai/evals/how-to/retry-strategies)
- [Dataset Management](/docs/ai/evals/how-to/dataset-management)
- [Logfire Integration](/docs/ai/evals/how-to/logfire-integration)

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/how-to/concurrency
