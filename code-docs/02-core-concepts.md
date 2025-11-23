# Core Concepts

This document explains the fundamental concepts and abstractions in Guardrails. Understanding these concepts is essential for working with the codebase.

## 🛡️ Guards

### What is a Guard?

A `Guard` is the primary interface for validation in Guardrails. It's a container that:
- Composes multiple validators
- Manages validation execution
- Handles LLM integration
- Tracks execution history

### Guard Class Hierarchy

```
IGuard (Interface from guardrails-api-client)
  ↓
Guard (guardrails/guard.py)
  ├─ Synchronous execution
  ├─ Schema management
  └─ Validator composition

AsyncGuard (guardrails/async_guard.py)
  └─ Async/await support
```

### Creating Guards

**From Pydantic Models:**
```python
from pydantic import BaseModel
from guardrails import Guard

class Pet(BaseModel):
    name: str
    age: int

guard = Guard.for_pydantic(output_class=Pet)
```

**From JSON Schema:**
```python
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer"}
    }
}

guard = Guard.from_dict(schema)
```

**From RAIL (Guardrails Spec):**
```python
guard = Guard.from_rail("spec.rail")
# or
guard = Guard.from_rail_string("<rail>...</rail>")
```

**With Validators:**
```python
from guardrails import Guard
from guardrails.hub import RegexMatch, ToxicLanguage

guard = Guard().use(
    RegexMatch(regex=r"\d{3}-\d{4}")
)

# Multiple validators
guard = Guard().use_many(
    RegexMatch(regex=r"\d{3}-\d{4}"),
    ToxicLanguage(threshold=0.5)
)
```

### Guard Configuration

Guards support various configuration options:

```python
guard = Guard(
    name="my_guard",              # Identifier for the guard
    description="Validates...",   # Human-readable description
    num_reasks=2,                 # Max retry attempts
    reask_prompt="Please fix...", # Custom reask message
    reask_messages="...",         # For chat-based models
)
```

### Guard Methods

**Validation Only:**
```python
# Validate raw data
result = guard.validate(value)

# Parse and validate
result = guard.parse(
    llm_output="...",
    metadata={...}
)
```

**With LLM Integration:**
```python
# Call LLM and validate
result = guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate...",
    model="gpt-4",
    **kwargs
)
```

**Streaming:**
```python
# Stream with validation
for fragment in guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate...",
    stream=True
):
    print(fragment.validated_output)
```

## ✅ Validators

### What is a Validator?

A `Validator` is a reusable validation component that:
- Checks data against specific criteria
- Returns validation results (pass/fail)
- Can operate on different data types
- Supports various validation scopes

### Validator Base Class

Located in `guardrails/validator_base.py`:

```python
@dataclass
class Validator:
    """Base class for validators."""
    
    # Configuration
    rail_alias: str = ""                    # Name in RAIL format
    run_in_separate_process = False         # Isolate execution
    override_value_on_pass = False          # Can modify value
    
    # Core method
    def validate(self, value, metadata) -> ValidationResult:
        """Main validation logic."""
        pass
```

### Validation Results

Validators return one of:

**PassResult:**
```python
from guardrails import PassResult

return PassResult()
# Or with modified value
return PassResult(value_override=new_value)
```

**FailResult:**
```python
from guardrails import FailResult

return FailResult(
    error_message="Validation failed because...",
    fix_value=corrected_value,  # Optional: suggested fix
)
```

### Validation Scopes

Validators can operate at different scopes:

```python
class MyValidator(Validator):
    def validate(self, value, metadata):
        """Validates the entire value."""
        pass
    
    def validate_each(self, value, metadata):
        """Validates each element in a list."""
        pass
    
    def validate_stream(self, chunk, metadata):
        """Validates streaming chunks."""
        pass
```

### Validator Types

**1. Built-in Validators** (`guardrails/validators/`)

Simple validators included with Guardrails:
- String validators
- Numeric validators
- Structural validators

**2. Hub Validators** (`guardrails/hub/`)

Downloaded from Guardrails Hub:
- Advanced ML-based validators
- Domain-specific validators
- Community-contributed validators

**3. Custom Validators**

User-defined validators:

```python
from guardrails import Validator, register_validator
from guardrails import PassResult, FailResult

@register_validator(name="even_number", data_type="integer")
class EvenNumber(Validator):
    def validate(self, value, metadata):
        if value % 2 == 0:
            return PassResult()
        return FailResult(
            error_message=f"{value} is not even"
        )
```

### Validator Configuration

Validators accept configuration parameters:

```python
from guardrails.hub import RegexMatch

validator = RegexMatch(
    regex=r"\d{3}-\d{4}",        # Required parameter
    match_type="fullmatch",      # Optional parameter
    on_fail=OnFailAction.REASK   # Failure handling
)
```

### Validator Registration

Validators are registered for discovery:

```python
@register_validator(
    name="my_validator",         # Unique identifier
    data_type="string",          # string, integer, object, etc.
)
class MyValidator(Validator):
    pass
```

## 🎬 OnFailAction

### What are OnFailActions?

`OnFailAction` defines what happens when validation fails. Located in `guardrails/types/on_fail.py`:

```python
class OnFailAction(str, Enum):
    REASK = "reask"           # Ask LLM to correct
    FIX = "fix"               # Attempt automatic fix
    FILTER = "filter"         # Remove invalid content
    REFRAIN = "refrain"       # Skip validation
    EXCEPTION = "exception"   # Raise ValidationError
    FIX_REASK = "fix_reask"  # Try fix, then reask
    CUSTOM = "custom"         # Custom function
```

### Action Behavior

**REASK:**
```python
# LLM is called again with error information
validator = SomeValidator(on_fail=OnFailAction.REASK)
```
- Constructs new prompt with error details
- Retries up to `num_reasks` times
- Useful for fixable errors

**FIX:**
```python
validator = SomeValidator(on_fail=OnFailAction.FIX)
```
- Validator attempts automatic correction
- Uses `fix_value` from FailResult
- No LLM call required

**FILTER:**
```python
validator = SomeValidator(on_fail=OnFailAction.FILTER)
```
- Removes invalid elements
- Continues with valid data
- Useful for lists/arrays

**REFRAIN:**
```python
validator = SomeValidator(on_fail=OnFailAction.REFRAIN)
```
- Skips validation
- Returns None or empty value
- Prevents blocking on non-critical checks

**EXCEPTION:**
```python
validator = SomeValidator(on_fail=OnFailAction.EXCEPTION)
```
- Raises `ValidationError`
- Stops execution immediately
- Useful for critical validations

**CUSTOM:**
```python
def custom_handler(value, fail_result):
    # Custom logic
    return corrected_value

validator = SomeValidator(
    on_fail=OnFailAction.CUSTOM,
    on_fail_handler=custom_handler
)
```

## 📋 Schemas

### Schema Types

Guardrails supports multiple schema formats:

**1. Pydantic Models:**
```python
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="Person's name")
    age: int = Field(ge=0, le=150)
    email: str = Field(pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$")

guard = Guard.for_pydantic(output_class=Person)
```

**2. JSON Schema:**
```python
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {
            "type": "integer",
            "minimum": 0,
            "maximum": 150
        }
    },
    "required": ["name", "age"]
}

guard = Guard.from_dict(schema)
```

**3. RAIL (Guardrails Spec):**
```xml
<rail version="0.1">
<output>
    <object name="person">
        <string name="name" description="Person's name"/>
        <integer name="age" description="Person's age" 
                 validators="is-in-range: min=0 max=150"/>
    </object>
</output>
</rail>
```

### Schema Processing

Schema processing flow (`schema/` module):

```
Input Schema (Pydantic/JSON/RAIL)
    ↓
Schema Parser (pydantic_schema.py, rail_schema.py)
    ↓
ProcessedSchema (internal representation)
    ↓
Prompt Generation (prompt_utils.py)
    ↓
LLM Instruction
```

### Schema Validation

Schemas are validated using `schema/validator.py`:

```python
from guardrails.schema.validator import validate_json_schema

# Validate schema is correct
validate_json_schema(schema)

# Validate data against schema
schema_validation(data, schema, validator_map)
```

## 🏃 Runners

### What is a Runner?

A `Runner` orchestrates the complete validation workflow:
- LLM API calls
- Response parsing
- Validator execution
- Retry/reask logic

Located in `guardrails/run/`:

```
runner.py          # Synchronous runner
async_runner.py    # Asynchronous runner
stream_runner.py   # Streaming runner
async_stream_runner.py  # Async streaming
```

### Runner Lifecycle

```
1. Initialize Runner
    ↓
2. Prepare Prompt (with schema)
    ↓
3. Call LLM API
    ↓
4. Parse Response
    ↓
5. Run Validators
    ↓
6. Check Results
    ├─ All Pass → Return Output
    └─ Some Fail → Handle OnFailAction
        ├─ Reask → Go to step 2
        ├─ Fix → Go to step 5
        └─ Other → Process and return
```

### Runner Configuration

Runners are created by Guards with:

```python
Runner(
    output_type=OutputTypes.DICT,      # Expected output type
    output_schema=schema,               # Validation schema
    num_reasks=2,                       # Max retries
    validation_map=validator_map,       # Validators to run
    messages=messages,                  # For chat models
    api=llm_api,                        # LLM callable
    metadata=metadata,                  # Context data
    exec_options=GuardExecutionOptions()
)
```

## 📊 Execution Context

### Call History

Every Guard execution creates a `Call` object (`classes/history/call.py`):

```python
class Call:
    inputs: CallInputs           # Original inputs
    iterations: List[Iteration]  # Each attempt
    outputs: Outputs             # Final outputs
    validator_logs: List[...]    # Validation results
```

### Iterations

Each retry creates an `Iteration`:

```python
class Iteration:
    inputs: Inputs               # Iteration inputs
    outputs: Outputs             # Iteration outputs
    validator_results: [...]     # Results for this iteration
    reask_prompts: [...]         # Generated reask prompts
```

### Metadata

Metadata flows through validation:

```python
metadata = {
    "request_id": "...",
    "user_id": "...",
    "custom_data": {...}
}

result = guard.validate(value, metadata=metadata)

# Validators can access metadata
class MyValidator(Validator):
    def validate(self, value, metadata):
        user_id = metadata.get("user_id")
        # Use metadata in validation
```

## 🔄 ValidationOutcome

### Structure

`ValidationOutcome` is the result of validation:

```python
class ValidationOutcome:
    raw_llm_output: str              # Original LLM response
    validated_output: Any            # Validated/corrected output
    reask: Optional[ReAsk]           # Reask information
    validation_passed: bool          # Overall success
    error: Optional[str]             # Error message if failed
    error_spans: List[ErrorSpan]     # Specific error locations
```

### Using ValidationOutcome

```python
result = guard.validate(data)

if result.validation_passed:
    print(f"Valid: {result.validated_output}")
else:
    print(f"Invalid: {result.error}")
    for span in result.error_spans:
        print(f"  - {span.reason}")
```

## 🎯 ValidatorReference

### What is ValidatorReference?

A `ValidatorReference` connects validators to specific schema locations:

```python
class ValidatorReference:
    id: str                    # Unique identifier
    validator_name: str        # Validator type
    on_fail: OnFailAction     # Failure handling
    args: Dict                 # Configuration
    on_path: str              # Schema path (e.g., "$.person.age")
```

### Validator Map

Guards maintain a `ValidatorMap`:

```python
validator_map = {
    "$.person.name": [
        ValidatorReference(validator_name="length", ...),
        ValidatorReference(validator_name="regex_match", ...)
    ],
    "$.person.age": [
        ValidatorReference(validator_name="is_integer", ...)
    ]
}
```

This maps schema paths to validators.

## 🔌 LLM Integration

### PromptCallableBase

Base class for LLM integrations (`llm_providers.py`):

```python
class PromptCallableBase:
    def __call__(self, *args, **kwargs):
        """Call the LLM API."""
        pass
```

### Supported LLM APIs

Guardrails works with:
- OpenAI (GPT-3.5, GPT-4, etc.)
- Anthropic (Claude)
- LiteLLM (unified interface)
- Custom APIs (via PromptCallableBase)
- Local models (via LiteLLM)

### LLM Response Handling

Responses are parsed based on output type:

```python
# For structured output
parsed = parse_llm_output(
    raw_output,
    output_schema,
    output_type=OutputTypes.DICT
)

# For string output
parsed = raw_output.strip()
```

## 🎨 Output Types

Guardrails supports different output types:

```python
class OutputTypes(str, Enum):
    STRING = "str"        # Plain text
    LIST = "list"         # List of items
    DICT = "dict"         # JSON object
    BOOLEAN = "bool"      # True/False
    INTEGER = "int"       # Numbers
    FLOAT = "float"       # Decimals
```

Output type determines:
- How LLM response is parsed
- Which validators can be applied
- Return type of Guard methods

## 📦 Hub Integration

### Validator Packages

Hub validators are Python packages:

```bash
guardrails hub install hub://guardrails/regex_match
```

This:
1. Downloads validator package
2. Installs dependencies
3. Makes validator available for import

### Using Hub Validators

```python
from guardrails.hub import RegexMatch

validator = RegexMatch(regex=r"\d{3}-\d{4}")
guard = Guard().use(validator)
```

### Remote Inference

Some validators use ML models:

```python
class MLValidator(Validator):
    def validate(self, value, metadata):
        # May call remote inference endpoint
        result = remote_inference(
            validator_name=self.rail_alias,
            value=value,
            model_params=self.get_model_params()
        )
        return result
```

## 🔍 Key Takeaways

1. **Guards** compose validators and manage execution
2. **Validators** implement reusable validation logic
3. **OnFailActions** define failure handling strategies
4. **Schemas** define expected structure
5. **Runners** orchestrate the complete workflow
6. **Context** flows through validation via metadata
7. **Hub** provides pre-built validators

## 📚 Next Steps

- [Workflow and Execution](./03-workflow-execution.md) - See how these concepts work together
- [API Structure](./04-api-structure.md) - Learn the public APIs
- [Module Reference](./07-module-reference.md) - Dive into specific implementations

---

Understanding these core concepts is essential for working effectively with Guardrails.
