---
type: Web Page
title: Dataset Serialization | Pydantic Docs
resource: https://pydantic.dev/docs/ai/evals/how-to/dataset-serialization
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Dataset Serialization

Learn how to save and load datasets in different formats, with support for custom evaluators and IDE integration.

Pydantic Evals supports serializing datasets to files in two formats:

- **YAML**(- `.yaml`,- `.yml`) - Human-readable, great for version control
- **JSON**(- `.json`) - Structured, machine-readable

Both formats support:

- Automatic JSON schema generation for IDE autocomplete and validation
- Custom evaluator serialization/deserialization
- Type-safe loading with generic parameters

YAML is the recommended format for most use cases due to its readability and compact syntax.

```
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected, IsInstance
# Create a dataset with typed parameters
dataset = Dataset[str, str, Any](
    name='my_tests',
    cases=[
        Case(
            name='test_1',
            inputs='hello',
            expected_output='HELLO',
        ),
    ],
    evaluators=[
        IsInstance(type_name='str'),
        EqualsExpected(),
    ],
)
# Save to YAML
dataset.to_file('my_tests.yaml')
```
This creates two files:

- `my_tests.yaml`
- `my_tests_schema.json`

```
# yaml-language-server: $schema=my_tests_schema.json
name: my_tests
cases:
- name: test_1
  inputs: hello
  expected_output: HELLO
evaluators:
- IsInstance: str
- EqualsExpected
```
The first line references the schema file:

```
# yaml-language-server: $schema=my_tests_schema.json
```
This enables:

- ✅ **Autocomplete**in VS Code, PyCharm, and other editors
- ✅ **Inline validation**while editing
- ✅ **Documentation tooltips**for fields
- ✅ **Error highlighting**for invalid data

```
from pathlib import Path
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected, IsInstance
# First create and save the dataset
Path('my_tests.yaml').parent.mkdir(exist_ok=True)
dataset = Dataset[str, str, Any](
    name='my_tests',
    cases=[Case(name='test_1', inputs='hello', expected_output='HELLO')],
    evaluators=[IsInstance(type_name='str'), EqualsExpected()],
)
dataset.to_file('my_tests.yaml')
# Load the dataset with type parameters
dataset = Dataset[str, str, Any].from_file('my_tests.yaml')
def my_task(text: str) -> str:
    return text.upper()
# Run evaluation
report = dataset.evaluate_sync(my_task)
```
JSON format is useful for programmatic generation or when strict structure is required.

```
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected
dataset = Dataset[str, str, Any](
    name='my_tests',
    cases=[
        Case(name='test_1', inputs='hello', expected_output='HELLO'),
    ],
    evaluators=[EqualsExpected()],
)
# Save to JSON
dataset.to_file('my_tests.json')
```
```
{
  "$schema": "my_tests_schema.json",
  "name": "my_tests",
  "cases": [
    {
      "name": "test_1",
      "inputs": "hello",
      "expected_output": "HELLO"
    }
  ],
  "evaluators": [
    "EqualsExpected"
  ]
}
```
The `$schema` key at the top enables IDE support similar to YAML.

```
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import EqualsExpected
# First create and save the dataset
dataset = Dataset[str, str, Any](
    name='my_tests',
    cases=[Case(name='test_1', inputs='hello', expected_output='HELLO')],
    evaluators=[EqualsExpected()],
)
dataset.to_file('my_tests.json')
# Load from JSON
dataset = Dataset[str, str, Any].from_file('my_tests.json')
```
By default, `to_file()` creates a JSON schema file alongside your dataset:

```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_tests', cases=[Case(inputs='test')])
# Creates both my_tests.yaml AND my_tests_schema.json
dataset.to_file('my_tests.yaml')
```
```
from pathlib import Path
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_tests', cases=[Case(inputs='test')])
# Create directories
Path('data').mkdir(exist_ok=True)
# Custom schema filename (relative to dataset file location)
dataset.to_file(
    'data/my_tests.yaml',
    schema_path='my_schema.json',
)
# No schema file
dataset.to_file('my_tests.yaml', schema_path=None)
```
Use `{stem}` to reference the dataset filename:

```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_tests', cases=[Case(inputs='test')])
# Creates: my_tests.yaml and my_tests.schema.json
dataset.to_file(
    'my_tests.yaml',
    schema_path='{stem}.schema.json',
)
```
Generate a schema without saving the dataset:

```
import json
from typing import Any
from pydantic_evals import Dataset
# Get schema as dictionary for a specific dataset type
schema = Dataset[str, str, Any].model_json_schema_with_evaluators()
# Save manually
with open('custom_schema.json', 'w', encoding='utf-8') as f:
    json.dump(schema, f, indent=2)
```
Custom evaluators require special handling during serialization and deserialization.

Custom evaluators must:

- Be decorated with `@dataclass`
- Inherit from `Evaluator`
- Be passed to both `to_file()`and`from_file()`

```
from dataclasses import dataclass
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class CustomThreshold(Evaluator):
    """Check if output length exceeds a threshold."""
    min_length: int
    max_length: int = 100
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        length = len(str(ctx.output))
        return self.min_length <= length <= self.max_length
# Create dataset with custom evaluator
dataset = Dataset[str, str, Any](
    name='custom_threshold_tests',
    cases=[
        Case(
            name='test_length',
            inputs='example',
            expected_output='long result',
            evaluators=[
                CustomThreshold(min_length=5, max_length=20),
            ],
        ),
    ],
)
# Save with custom evaluator types
dataset.to_file(
    'dataset.yaml',
    custom_evaluator_types=[CustomThreshold],
)
```
```
# yaml-language-server: $schema=dataset_schema.json
cases:
- name: test_length
  inputs: example
  expected_output: long result
  evaluators:
  - CustomThreshold:
      min_length: 5
      max_length: 20
```
```
from dataclasses import dataclass
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class CustomThreshold(Evaluator):
    """Check if output length exceeds a threshold."""
    min_length: int
    max_length: int = 100
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        length = len(str(ctx.output))
        return self.min_length <= length <= self.max_length
# First create and save the dataset
dataset = Dataset[str, str, Any](
    name='custom_threshold_tests',
    cases=[
        Case(
            name='test_length',
            inputs='example',
            expected_output='long result',
            evaluators=[CustomThreshold(min_length=5, max_length=20)],
        ),
    ],
)
dataset.to_file('dataset.yaml', custom_evaluator_types=[CustomThreshold])
# Load with custom evaluator registry
dataset = Dataset[str, str, Any].from_file(
    'dataset.yaml',
    custom_evaluator_types=[CustomThreshold],
)
```
Evaluators can be serialized in three forms:

```
evaluators:
- EqualsExpected
- IsInstance: str  # Using default parameter
```
```
evaluators:
- IsInstance: str
- Contains: "required text"
- MaxDuration: 2.0
```
```
evaluators:
- CustomThreshold:
    min_length: 5
    max_length: 20
- LLMJudge:
    rubric: "Response is accurate"
    model: "openai:gpt-5"
    include_input: true
```
| Feature | YAML | JSON | 
|---|---|---|
| Human readable | ✅ Excellent | ⚠️ Good | 
| Comments | ✅ Yes | ❌ No | 
| Compact | ✅ Yes | ⚠️ Verbose | 
| Machine parsing | ✅ Good | ✅ Excellent | 
| IDE support | ✅ Yes | ✅ Yes | 
| Version control | ✅ Clean diffs | ⚠️ Noisy diffs | 

**Recommendation**: Use YAML for most cases, JSON for programmatic generation.

Customize how your evaluator appears in serialized files:

```
from dataclasses import dataclass
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class VeryLongDescriptiveEvaluatorName(Evaluator):
    @classmethod
    def get_serialization_name(cls) -> str:
        return 'ShortName'
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
```
In YAML:

```
evaluators:
- ShortName  # Instead of VeryLongDescriptiveEvaluatorName
```
**Problem**: YAML file doesn’t show autocomplete

**Solutions**:

- 
**Check the schema path**in the first line of YAML:`# yaml-language-server: $schema=correct_schema_name.json`
- 
**Verify schema file exists**in the same directory
- 
**Restart the language server**in your IDE
- 
**Install YAML extension**(VS Code: “YAML” by Red Hat)

**Problem**: `ValueError: Unknown evaluator name: 'CustomEvaluator'`

**Solution**: Pass `custom_evaluator_types` when loading:

```
from dataclasses import dataclass
from typing import Any
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
@dataclass
class CustomEvaluator(Evaluator):
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
# First create and save with custom evaluator
dataset = Dataset[str, str, Any](
    name='custom_eval_tests',
    cases=[Case(inputs='test', evaluators=[CustomEvaluator()])],
)
dataset.to_file('tests.yaml', custom_evaluator_types=[CustomEvaluator])
# Load with custom evaluator types
dataset = Dataset[str, str, Any].from_file(
    'tests.yaml',
    custom_evaluator_types=[CustomEvaluator],  # Required!
)
```
**Problem**: `ValueError: Cannot infer format from extension`

**Solution**: Specify format explicitly:

```
from typing import Any
from pydantic_evals import Case, Dataset
dataset = Dataset[str, str, Any](name='my_tests', cases=[Case(inputs='test')])
# Explicit format for unusual extensions
dataset.to_file('data.txt', fmt='yaml')
dataset_loaded = Dataset[str, str, Any].from_file('data.txt', fmt='yaml')
```
**Problem**: Custom evaluator causes schema generation to fail

**Solution**: Ensure evaluator is a proper dataclass:

```
from dataclasses import dataclass
from pydantic_evals.evaluators import Evaluator, EvaluatorContext
# ✅ Correct
@dataclass
class MyEvaluator(Evaluator):
    value: int
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
# ❌ Wrong: Missing @dataclass
class BadEvaluator(Evaluator):
    def __init__(self, value: int):
        self.value = value
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return True
```
- [Dataset Management](/docs/ai/evals/how-to/dataset-management)
- [Custom Evaluators](/docs/ai/evals/evaluators/custom)
- [Core Concepts](/docs/ai/evals/core-concepts)

# Citations

1. Source page: https://pydantic.dev/docs/ai/evals/how-to/dataset-serialization
