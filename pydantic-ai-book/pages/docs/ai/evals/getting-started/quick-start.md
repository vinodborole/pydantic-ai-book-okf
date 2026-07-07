---
type: Web Page
title: Quick Start | Pydantic Docs
resource: https://pydantic.dev/docs/ai/evals/getting-started/quick-start
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Quick Start

**Pydantic Evals** is a powerful evaluation framework for systematically testing and evaluating AI systems, from simple LLM calls to complex multi-agent applications.

Pydantic Evals helps you:

- **Create test datasets**with type-safe structured inputs and expected outputs
- **Run evaluations**against your AI systems with automatic concurrency
- **Score results**using deterministic checks, LLM judges, or custom evaluators
- **Generate reports**with detailed metrics, assertions, and performance data
- **Track changes**by comparing evaluation runs over time
- **Integrate with Logfire**for visualization and collaborative analysis

For OpenTelemetry tracing and Logfire integration:

While evaluations are typically used to test AI systems, the Pydantic Evals framework works with any function call. To demonstrate the core functionality, we’ll start with a simple, deterministic example.

Here’s a complete example of evaluating a simple text transformation function:

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Contains, EqualsExpected
# Create a dataset with test cases
dataset = Dataset(
    name='uppercase_tests',
    cases=[
        Case(
            name='uppercase_basic',
            inputs='hello world',
            expected_output='HELLO WORLD',
        ),
        Case(
            name='uppercase_with_numbers',
            inputs='hello 123',
            expected_output='HELLO 123',
        ),
    ],
    evaluators=[
        EqualsExpected(),  # Check exact match with expected_output
        Contains(value='HELLO', case_sensitive=True),  # Check contains "HELLO"
    ],
)
# Define the function to evaluate
def uppercase_text(text: str) -> str:
    return text.upper()
# Run the evaluation
report = dataset.evaluate_sync(uppercase_text)
# Print the results
report.print()
"""
        Evaluation Summary: uppercase_text
┏━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━┓
┃ Case ID                ┃ Assertions ┃ Duration ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━┩
│ uppercase_basic        │ ✔✔         │     10ms │
├────────────────────────┼────────────┼──────────┤
│ uppercase_with_numbers │ ✔✔         │     10ms │
├────────────────────────┼────────────┼──────────┤
│ Averages               │ 100.0% ✔   │     10ms │
└────────────────────────┴────────────┴──────────┘
"""
```
Output:

```
                  Evaluation Summary: uppercase_text
┏━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━┓
┃ Case ID                 ┃ Assertions ┃ Duration ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━┩
│ uppercase_basic         │ ✔✔         │     10ms │
├─────────────────────────┼────────────┼──────────┤
│ uppercase_with_numbers  │ ✔✔         │     10ms │
├─────────────────────────┼────────────┼──────────┤
│ Averages                │ 100.0% ✔   │     10ms │
└─────────────────────────┴────────────┴──────────┘
```
Understanding a few core concepts will help you get the most out of Pydantic Evals:

- `Dataset`
- `Case`
- `Evaluator`
- `EvaluationReport`

For a deeper dive, see Core Concepts.

Test that your AI system produces correctly-structured outputs:

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Contains, IsInstance
dataset = Dataset(
    name='dict_validation',
    cases=[
        Case(inputs={'data': 'required_key present'}, expected_output={'result': 'success'}),
    ],
    evaluators=[
        IsInstance(type_name='dict'),
        Contains(value='required_key'),
    ],
)
```
Use an LLM to evaluate subjective qualities like accuracy or helpfulness:

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import LLMJudge
dataset = Dataset(
    name='llm_judge_test',
    cases=[
        Case(inputs='What is the capital of France?', expected_output='Paris'),
    ],
    evaluators=[
        LLMJudge(
            rubric='Response is accurate and helpful',
            include_input=True,
            model='anthropic:claude-sonnet-4-6',
        )
    ],
)
```
Ensure your system meets performance requirements:

```
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import MaxDuration
dataset = Dataset(
    name='performance_test',
    cases=[
        Case(inputs='test input', expected_output='test output'),
    ],
    evaluators=[
        MaxDuration(seconds=2.0),
    ],
)
```
Explore the documentation to learn more:

- **Core Concepts**- Understand the data model and evaluation flow
- **Native Evaluators**- Learn about all available evaluators
- **Custom Evaluators**- Write your own evaluation logic
- **Dataset Management**- Save, load, and generate datasets
- **Examples**- Practical examples for common scenarios

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/getting-started/quick-start
