---
type: Web Page
title: Dataset Management | Pydantic Docs
resource: https://pydantic.dev/docs/ai/evals/how-to/dataset-management
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Dataset Management

Create, save, load, and generate evaluation datasets.

Define datasets directly in Python:

```
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected, IsInstance
dataset = Dataset[str, str, Any](
    name='my_eval_suite',
    cases=[
        Case(
            name='test_1',
            inputs='input 1',
            expected_output='output 1',
        ),
        Case(
            name='test_2',
            inputs='input 2',
            expected_output='output 2',
        ),
    ],
    evaluators=[
        IsInstance(type_name='str'),
        EqualsExpected(),
    ],
)
```
```
from typing import Any
from pydantic_evals import Dataset
from pydantic_evals.evaluators import IsInstance
dataset = Dataset[str, str, Any](name='dynamic_dataset', cases=[], evaluators=[])
# Add cases one at a time
dataset.add_case(
    name='dynamic_case',
    inputs='test input',
    expected_output='test output',
)
# Add evaluators
dataset.add_evaluator(IsInstance(type_name='str'))
```
```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_eval_suite', cases=[Case(name='test', inputs='example')])
dataset.to_file('my_dataset.yaml')
# Also saves schema file: my_dataset_schema.json
```
Output (`my_dataset.yaml`):

```
# yaml-language-server: $schema=my_dataset_schema.json
name: my_eval_suite
cases:
- name: test_1
  inputs: input 1
  expected_output: output 1
  evaluators:
  - EqualsExpected
- name: test_2
  inputs: input 2
  expected_output: output 2
  evaluators:
  - EqualsExpected
evaluators:
- IsInstance: str
```
```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_eval_suite', cases=[Case(name='test', inputs='example')])
dataset.to_file('my_dataset.json')
# Also saves schema file: my_dataset_schema.json
```
```
from pathlib import Path
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_eval_suite', cases=[Case(name='test', inputs='example')])
# Custom schema location
Path('data').mkdir(exist_ok=True)
Path('data/schemas').mkdir(parents=True, exist_ok=True)
dataset.to_file(
    'data/my_dataset.yaml',
    schema_path='schemas/my_schema.json',
)
# No schema file
dataset.to_file('my_dataset.yaml', schema_path=None)
```
```
from typing import Any
from pydantic_evals import Dataset
# Infers format from extension
dataset = Dataset[str, str, Any].from_file('my_dataset.yaml')
dataset = Dataset[str, str, Any].from_file('my_dataset.json')
# Explicit format for non-standard extensions
dataset = Dataset[str, str, Any].from_file('data.txt', fmt='yaml')
```
```
from typing import Any
from pydantic_evals import Dataset
yaml_content = """
name: my_tests
cases:
- name: test
  inputs: hello
  expected_output: HELLO
evaluators:
- EqualsExpected
"""
dataset = Dataset[str, str, Any].from_text(yaml_content, fmt='yaml')
```
```
from typing import Any
from pydantic_evals import Dataset
data = {
    'name': 'my_tests',
    'cases': [
        {
            'name': 'test',
            'inputs': 'hello',
            'expected_output': 'HELLO',
        },
    ],
    'evaluators': [{'EqualsExpected': {}}],
}
dataset = Dataset[str, str, Any].from_dict(data)
```
When loading datasets that use custom evaluators, you must pass them to `from_file()`:

```
from dataclasses import dataclass
from typing import Any
from pydantic_evals import Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class MyCustomEvaluator(Evaluator):
    threshold: float = 0.5
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
# Load with custom evaluator registry
dataset = Dataset[str, str, Any].from_file(
    'my_dataset.yaml',
    custom_evaluator_types=[MyCustomEvaluator],
)
```
For complete details on serialization with custom evaluators, see [Dataset Serialization](/docs/ai/evals/how-to/dataset-serialization).

Pydantic Evals allows you to generate test datasets using LLMs with [ generate_dataset](/docs/ai/api/pydantic_evals/generation/#pydantic_evals.generation.generate_dataset).

Datasets can be generated in either JSON or YAML format, in both cases a JSON schema file is generated alongside the dataset and referenced in the dataset, so you should get type checking and auto-completion in your editor.

Define the schema for the inputs to the task.

Define the schema for the expected outputs of the task.

Define the schema for the metadata of the test cases.

Call [ generate_dataset](/docs/ai/api/pydantic_evals/generation/#pydantic_evals.generation.generate_dataset) to create a 

[with 2 cases confirming to the schema.](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset)

`Dataset`Save the dataset to a YAML file, this will also write `questions_cases_schema.json` with the schema JSON schema for `questions_cases.yaml` to make editing easier. The magic `yaml-language-server` comment is supported by at least vscode, jetbrains/pycharm (more details [here](https://github.com/redhat-developer/yaml-language-server#using-inlined-schema)).

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main(answer)) to run main)*

You can also write datasets as JSON files:

Generate the [ Dataset](/docs/ai/api/pydantic_evals/dataset/#pydantic_evals.dataset.Dataset) exactly as above.

Save the dataset to a JSON file, this will also write `questions_cases_schema.json` with the JSON schema for `questions_cases.json`. This time the `$schema` key is included in the JSON file to define the schema for IDEs to use while you edit the file, there's no formal spec for this, but it works in vscode and pycharm and is discussed at length in [json-schema-org/json-schema-spec#828](https://github.com/json-schema-org/json-schema-spec/issues/828).

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main(answer)) to run main)*

Use generic type parameters for type safety:

```
from typing_extensions import TypedDict
from pydantic_evals import Case, Dataset
class MyInput(TypedDict):
    query: str
    max_results: int
class MyOutput(TypedDict):
    results: list[str]
class MyMetadata(TypedDict):
    category: str
# Type-safe dataset
dataset: Dataset[MyInput, MyOutput, MyMetadata] = Dataset(
    name='typed_dataset',
    cases=[
        Case(
            name='test',
            inputs={'query': 'test', 'max_results': 10},
            expected_output={'results': ['a', 'b']},
            metadata={'category': 'search'},
        ),
    ],
)
```
Generate JSON Schema for IDE support:

```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_eval_suite', cases=[Case(name='test', inputs='example')])
# Save with schema
dataset.to_file('my_dataset.yaml')  # Creates my_dataset_schema.json
# Schema enables:
# - Autocomplete in VS Code/PyCharm
# - Validation while editing
# - Inline documentation
```
Manual schema generation:

```
import json
from dataclasses import dataclass
from typing import Any
from pydantic_evals import Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class MyCustomEvaluator(Evaluator):
    threshold: float = 0.5
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
schema = Dataset[str, str, Any].model_json_schema_with_evaluators(
    custom_evaluator_types=[MyCustomEvaluator],
)
print(json.dumps(schema, indent=2)[:66] + '...')
"""
{
  "$defs": {
    "Case": {
      "additionalProperties": false,
...
"""
```
```
from pydantic_evals import Case
# Good
Case(name='uppercase_basic_ascii', inputs='hello')
Case(name='uppercase_unicode_emoji', inputs='hello 😀')
Case(name='uppercase_empty_string', inputs='')
# Bad
Case(name='test1', inputs='hello')
Case(name='test2', inputs='world')
Case(name='test3', inputs='foo')
```
```
from pydantic_evals import Case, Dataset
dataset = Dataset(
    name='organized_by_difficulty',
    cases=[
        Case(name='easy_1', inputs='test', metadata={'difficulty': 'easy'}),
        Case(name='easy_2', inputs='test2', metadata={'difficulty': 'easy'}),
        Case(name='medium_1', inputs='test3', metadata={'difficulty': 'medium'}),
        Case(name='hard_1', inputs='test4', metadata={'difficulty': 'hard'}),
    ],
)
```
```
from pydantic_evals import Case, Dataset
# Start with representative cases
dataset = Dataset(
    name='starting_small',
    cases=[
        Case(name='happy_path', inputs='test'),
        Case(name='edge_case', inputs=''),
        Case(name='error_case', inputs='invalid'),
    ],
)
# Add more as you find issues
dataset.add_case(name='newly_discovered_edge_case', inputs='edge')
```
Case-specific evaluators let different cases have different evaluation criteria, which is essential for comprehensive “test coverage”. Rather than trying to write one-size-fits-all evaluators, you can specify exactly what “good” looks like for each scenario. This is particularly powerful with [ LLMJudge](/docs/ai/api/pydantic_evals/evaluators/#pydantic_evals.evaluators.LLMJudge) evaluators where you can describe nuanced requirements per case, making it easy to build and maintain golden datasets. See 

[Case-specific evaluators](/docs/ai/evals/evaluators/overview#case-specific-evaluators)for detailed guidance.

```
from typing import Any
from pydantic_evals import Case, Dataset
# First create some test datasets
for name in ['smoke_tests', 'comprehensive_tests', 'regression_tests']:
    test_dataset = Dataset[str, Any, Any](name=name, cases=[Case(name='test', inputs='example')])
    test_dataset.to_file(f'{name}.yaml')
# Smoke tests (fast, critical paths)
smoke_tests = Dataset[str, Any, Any].from_file('smoke_tests.yaml')
# Comprehensive tests (slow, thorough)
comprehensive = Dataset[str, Any, Any].from_file('comprehensive_tests.yaml')
# Regression tests (specific bugs)
regression = Dataset[str, Any, Any].from_file('regression_tests.yaml')
```
- [Dataset Serialization](/docs/ai/evals/how-to/dataset-serialization)
- [Generating Datasets](#generating-datasets)
- [Examples: Simple Validation](/docs/ai/evals/examples/simple-validation)

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/how-to/dataset-management
