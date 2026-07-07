---
type: Web Page
title: Multi-Run Evaluation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/evals/how-to/multi-run
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Multi-Run Evaluation

Run each case multiple times to measure variability and get more reliable aggregate results.

AI systems are inherently stochastic — the same input can produce different outputs across runs. The `repeat` parameter lets you run each case multiple times and automatically aggregates the results, giving you a clearer picture of your system’s typical behavior.

Pass `repeat` to `evaluate()` or `evaluate_sync()`:

```
from pydantic_evals import Case, Dataset
dataset = Dataset(
    name='multi_run_basic',
    cases=[
        Case(name='greeting', inputs='Say hello'),
        Case(name='farewell', inputs='Say goodbye'),
    ]
)
def task(inputs: str) -> str:
    return inputs.upper()
# Run each case 5 times
report = dataset.evaluate_sync(task, repeat=5)
# 2 cases × 5 repeats = 10 total runs
print(len(report.cases))
#> 10
```
When `repeat > 1`, each run gets an indexed name like `greeting [1/5]`, `greeting [2/5]`, etc., while the original case name is preserved in `source_case_name` for grouping.

Use `case_groups()` to access runs organized by original case, with per-group aggregated statistics:

```
from pydantic_evals import Case, Dataset
dataset = Dataset(
    name='grouped_results',
    cases=[
        Case(name='greeting', inputs='Say hello'),
        Case(name='farewell', inputs='Say goodbye'),
    ]
)
def task(inputs: str) -> str:
    return inputs.upper()
report = dataset.evaluate_sync(task, repeat=3)
groups = report.case_groups()
assert groups is not None  # None for single-run (repeat=1)
print(len(groups))
#> 2
group_names = [g.name for g in groups]
print(group_names)
#> ['greeting', 'farewell']
# Each group has 3 runs and aggregated statistics
for group in groups:
    assert len(group.runs) == 3
    assert len(group.failures) == 0
    assert group.summary.task_duration > 0
```
Each `ReportCaseGroup` contains:

- `name`— the original case name
- `runs`— the individual- `ReportCase`results
- `failures`— any runs that raised exceptions
- `summary`— a- `ReportCaseAggregate`with averaged scores, metrics, labels, assertions, and durations

With `repeat > 1`, the report’s `averages()` uses a two-level aggregation strategy:

- **Per-group averages**: Each case’s runs are averaged into a group summary
- **Cross-group averages**: The group summaries are averaged to produce the final result

This ensures each original case contributes equally to the overall averages, regardless of how many runs succeeded or failed.

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected
dataset = Dataset(
    name='aggregation',
    cases=[
        Case(name='easy', inputs='hello', expected_output='HELLO'),
        Case(name='hard', inputs='world', expected_output='WORLD'),
    ],
    evaluators=[EqualsExpected()],
)
def task(inputs: str) -> str:
    return inputs.upper()
report = dataset.evaluate_sync(task, repeat=3)
averages = report.averages()
assert averages is not None
print(f'Overall assertion rate: {averages.assertions}')
#> Overall assertion rate: 1.0
```
When `repeat=1` (the default), behavior is identical to a standard evaluation — no run indexing, no `source_case_name`, and `case_groups()` returns `None`:

```
from pydantic_evals import Case, Dataset
dataset = Dataset(name='default_behavior', cases=[Case(name='test', inputs='hello')])
def task(inputs: str) -> str:
    return inputs.upper()
report = dataset.evaluate_sync(task)  # repeat=1 by default
assert report.case_groups() is None
assert all(c.source_case_name is None for c in report.cases)
```
- **Concurrency & Performance**— Control parallel execution with- `max_concurrency`
- **Metrics & Attributes**— Track custom metrics across runs
- **Logfire Integration**— Visualize multi-run results

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/how-to/multi-run
