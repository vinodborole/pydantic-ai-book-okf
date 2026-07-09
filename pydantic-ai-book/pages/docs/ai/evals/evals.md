---
type: Web Page
title: Pydantic Evals
resource: https://pydantic.dev/docs/ai/evals/evals
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Overview

**Pydantic Evals** is a powerful evaluation framework for systematically testing and evaluating AI systems, from simple LLM calls to complex multi-agent applications.

**Getting Started:**

**Evaluators:**

- [Evaluators Overview](/docs/ai/evals/evaluators/overview)- Compare evaluator types and learn when to use each approach
- [Built-in Evaluators](/docs/ai/evals/evaluators/built-in)- Complete reference for exact match, instance checks, and other ready-to-use evaluators
- [LLM as a Judge](/docs/ai/evals/evaluators/llm-judge)- Use LLMs to evaluate subjective qualities, complex criteria, and natural language outputs
- [Custom Evaluators](/docs/ai/evals/evaluators/custom)- Implement domain-specific scoring logic and custom evaluation metrics
- [Span-Based Evaluation](/docs/ai/evals/evaluators/span-based)- Evaluate internal agent behavior (tool calls, execution flow) using OpenTelemetry traces. Essential for complex agents where correctness depends on- *how*the answer was reached, not just the final output. Also ensures eval assertions align with production telemetry.

**How-To Guides:**

- [Logfire Integration](/docs/ai/evals/how-to/logfire-integration)- Visualize results
- [Dataset Management](/docs/ai/evals/how-to/dataset-management)- Save, load, generate
- [Concurrency & Performance](/docs/ai/evals/how-to/concurrency)- Control parallel execution
- [Retry Strategies](/docs/ai/evals/how-to/retry-strategies)- Handle transient failures
- [Metrics & Attributes](/docs/ai/evals/how-to/metrics-attributes)- Track custom data
- [Case Lifecycle Hooks](/docs/ai/evals/how-to/lifecycle)- Per-case setup, teardown, and context enrichment

**Examples:**

- [Simple Validation](/docs/ai/evals/examples/simple-validation)- Basic example

**Reference:**

Pydantic Evals follows a **code-first approach** where you define all evaluation components (datasets, experiments, tasks, cases and evaluators) in Python code, or as serialized data loaded by Python code. This differs from platforms with fully web-based configuration.

When you run an *Experiment* you’ll see a progress indicator and can print the results wherever you run your python code (IDE, terminal, etc). You also get a report object back that you can serialize and store or send to a notebook or other application for further visualization and analysis.

If you are using [Pydantic Logfire](https://logfire.pydantic.dev/docs/guides/web-ui/evals/), your experiment results automatically appear in the Logfire web interface for visualization, comparison, and collaborative analysis. Logfire serves as a observability layer - you write and run evals in code, then view and analyze results in the web UI.

To install the Pydantic Evals package, run:

`pydantic-evals` does not depend on `pydantic-ai`, but has an optional dependency on `logfire` if you’d like to
use OpenTelemetry traces in your evals, or send evaluation results to [logfire](https://pydantic.dev/logfire).

Pydantic Evals is built around a simple data model:

```
Dataset (1) ──────────── (Many) Case
│                        │
│                        │
└─── (Many) Experiment ──┴─── (Many) Case results
     │
     └─── (1) Task
     │
     └─── (Many) Evaluator
```
- **Dataset → Cases**: One Dataset contains many Cases
- **Dataset → Experiments**: One Dataset can be used across many Experiments over time
- **Experiment → Case results**: One Experiment generates results by executing each Case
- **Experiment → Task**: One Experiment evaluates one defined Task
- **Experiment → Evaluators**: One Experiment uses multiple Evaluators. Dataset-wide Evaluators are run against all Cases, and Case-specific Evaluators against their respective Cases

- **Dataset creation**: Define cases and evaluators in YAML/JSON, or directly in Python
- **Experiment execution**: Run- `dataset.evaluate_sync(task_function)`
- **Cases run**: Each Case is executed against the Task
- **Evaluation**: Evaluators score the Task outputs for each Case
- **Results**: All Case results are collected into a summary report

For a deeper understanding, see [Core Concepts](/docs/ai/evals/core-concepts).

In Pydantic Evals, everything begins with [ Dataset](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset)s and 

[s:](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Case)

`Case`- `Dataset`
- `Case`

*(This example is complete, it can be run “as is”)*

See [Dataset Management](/docs/ai/evals/how-to/dataset-management) to learn about saving, loading, and generating datasets.

[ Evaluator](/docs/ai/api/pydantic_evals/evaluators/#pydantic_evals.evaluators.Evaluator)s analyze and score the results of your Task when tested against a Case.

These can be deterministic, code-based checks (such as testing model output format with a regex, or checking for the appearance of PII or sensitive data), or they can assess non-deterministic model outputs for qualities like accuracy, precision/recall, hallucinations, or instruction-following.

While both kinds of testing are useful in LLM systems, classical code-based tests are cheaper and easier than tests which require either human or machine review of model outputs.

Pydantic Evals includes several [built-in evaluators](/docs/ai/evals/evaluators/built-in) and allows you to define [custom evaluators](/docs/ai/evals/evaluators/custom):

You can add built-in evaluators to a dataset using the [ add_evaluator](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset.add_evaluator) method.

This custom evaluator returns a simple score based on whether the output matches the expected output.

*(This example is complete, it can be run “as is”)*

Learn more:

- [Evaluators Overview](/docs/ai/evals/evaluators/overview)- When to use different types
- [Built-in Evaluators](/docs/ai/evals/evaluators/built-in)- Complete reference
- [LLM Judge](/docs/ai/evals/evaluators/llm-judge)- Using LLMs as evaluators
- [Custom Evaluators](/docs/ai/evals/evaluators/custom)- Write your own logic
- [Span-Based Evaluation](/docs/ai/evals/evaluators/span-based)- Analyze execution traces

Performing evaluations involves running a task against all cases in a dataset, also known as running an “experiment”.

Putting the above two examples together and using the more declarative `evaluators` kwarg to [ Dataset](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset):

Create a [test case](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Case) as above

Create a [ Dataset](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset) with test cases and 

`evaluators`Our function to evaluate.

Run the evaluation with [ evaluate_sync](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset.evaluate_sync), which runs the function against all test cases in the dataset, and returns an 

[object.](/docs/ai/api/pydantic_evals/reporting/#pydantic_evals.reporting.EvaluationReport)

`EvaluationReport`Print the report with [ print](/docs/ai/api/pydantic_evals/reporting/#pydantic_evals.reporting.EvaluationReport.print), which shows the results of the evaluation. We have omitted duration here just to keep the printed output from changing from run to run.

*(This example is complete, it can be run “as is”)*

See [Quick Start](/docs/ai/evals/quick-start) for more examples and [Concurrency & Performance](/docs/ai/evals/how-to/concurrency) to learn about controlling parallel execution.

For comprehensive coverage of all classes, methods, and configuration options, see the detailed [API Reference documentation](https://ai.pydantic.dev/api/pydantic_evals/dataset/).

- **Start with simple evaluations**using- [Quick Start](/docs/ai/evals/quick-start)
- **Understand the data model**with- [Core Concepts](/docs/ai/evals/core-concepts)
- **Explore built-in evaluators**in- [Built-in Evaluators](/docs/ai/evals/evaluators/built-in)
- **Integrate with Logfire**for visualization:- [Logfire Integration](/docs/ai/evals/how-to/logfire-integration)
- **Build comprehensive test suites**with- [Dataset Management](/docs/ai/evals/how-to/dataset-management)
- **Implement custom evaluators**for domain-specific metrics:- [Custom Evaluators](/docs/ai/evals/evaluators/custom)

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/evals
