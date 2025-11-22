# API Structure

This document describes the public APIs, entry points, and how to use Guardrails programmatically.

## 📦 Public API Overview

The main entry points are exported from `guardrails/__init__.py`:

```python
from guardrails import (
    Guard,              # Main Guard class
    AsyncGuard,         # Async Guard
    Validator,          # Base validator class
    OnFailAction,       # Failure handling enum
    ValidationOutcome,  # Result type
    register_validator, # Validator registration
    settings,           # Configuration
    install,            # Hub installation
    # ... and more
)
```

## 🛡️ Guard API

### Creating Guards

**From Pydantic:**
```python
from pydantic import BaseModel
from guardrails import Guard

class Output(BaseModel):
    field: str

guard = Guard.for_pydantic(
    output_class=Output,
    prompt="Generate...",           # Optional
    instructions="...",             # Optional
    reask_prompt="Please fix...",   # Optional
    reask_instructions="..."        # Optional
)
```

**From String (primitive type):**
```python
guard = Guard.for_string(
    validators=[validator1, validator2],
    description="A string value"
)
```

**From Dict (JSON Schema):**
```python
schema = {"type": "object", "properties": {...}}
guard = Guard.from_dict(schema)
```

**From RAIL:**
```python
guard = Guard.from_rail("spec.rail")
# or
guard = Guard.from_rail_string("<rail>...</rail>")
```

**Empty Guard (add validators later):**
```python
guard = Guard()
guard.use(validator1)
guard.use_many(validator2, validator3)
```

### Guard Configuration

```python
guard = Guard(
    # Identification
    name="my_guard",
    description="Validates...",
    
    # Reask configuration
    num_reasks=2,
    reask_prompt="...",
    reask_messages=[...],
    
    # Output configuration
    output_schema={...},
    
    # Execution options
    tracer=custom_tracer,
)
```

### Validation Methods

**validate() - Validate raw value:**
```python
outcome = guard.validate(
    value="some text",
    metadata={"key": "value"},
    llm_api=None,          # Optional
    **llm_api_kwargs
)
```

**parse() - Parse and validate LLM output:**
```python
outcome = guard.parse(
    llm_output='{"field": "value"}',
    metadata={...},
    num_reasks=1,
    prompt_params={...},
    full_schema_reask=False
)
```

**__call__() - Full LLM integration:**
```python
outcome = guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate...",                    # For completion models
    messages=[...],                          # For chat models
    prompt_params={...},                     # Template variables
    num_reasks=None,                        # Override default
    metadata={...},                         # Context data
    full_schema_reask=False,                # Reask full/partial schema
    stream=False,                           # Enable streaming
    **llm_api_kwargs                        # Passed to LLM API
)
```

### Streaming

```python
# Streaming with validation
for fragment in guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate...",
    stream=True,
    **kwargs
):
    # Fragment is a ValidationOutcome
    print(fragment.validated_output)
    
    if fragment.validation_passed:
        print("✓ Valid")
    else:
        print(f"✗ Errors: {fragment.error}")
```

### Validator Management

**Adding validators:**
```python
# Single validator
guard.use(
    validator_class,
    on_fail=OnFailAction.EXCEPTION,
    **validator_kwargs
)

# Multiple validators
guard.use_many(
    validator1,
    validator2,
    validator3
)
```

**Validator configuration:**
```python
from guardrails.hub import RegexMatch

guard.use(
    RegexMatch,
    regex=r"\d{3}-\d{4}",
    match_type="fullmatch",
    on_fail=OnFailAction.REASK
)
```

### History Access

```python
# Get last call
last_call = guard.history.last

# Access iterations
for iteration in last_call.iterations:
    print(iteration.outputs)

# Get all validator logs
logs = last_call.validator_logs

# Check if validation passed
passed = last_call.outputs.validation_passed
```

### Guard Serialization

**To/From Dict:**
```python
# Serialize
guard_dict = guard.to_dict()

# Deserialize
guard = Guard.from_dict(guard_dict)
```

**Server Configuration:**
```python
# For server mode
config_dict = guard.to_config()
# Returns configuration suitable for server
```

## 🔄 AsyncGuard API

Async version with same interface:

```python
from guardrails import AsyncGuard

guard = AsyncGuard().use(validator)

# Async validate
outcome = await guard.validate(value)

# Async with LLM
outcome = await guard(
    llm_api=async_llm_function,
    prompt="..."
)

# Async streaming
async for fragment in guard(
    llm_api=async_llm_function,
    prompt="...",
    stream=True
):
    print(fragment.validated_output)
```

## ✅ Validator API

### Base Class

```python
from guardrails import Validator, register_validator
from guardrails.classes import PassResult, FailResult

@register_validator(name="my_validator", data_type="string")
class MyValidator(Validator):
    """Validator description."""
    
    def __init__(self, param1: str, on_fail: OnFailAction = None):
        super().__init__(on_fail=on_fail)
        self.param1 = param1
    
    def validate(self, value: Any, metadata: Dict) -> ValidationResult:
        """
        Main validation logic.
        
        Args:
            value: The value to validate
            metadata: Additional context
            
        Returns:
            PassResult or FailResult
        """
        if self.is_valid(value):
            return PassResult()
        
        return FailResult(
            error_message="Value is invalid",
            fix_value=self.suggest_fix(value)  # Optional
        )
```

### Validation Methods

**validate() - Main method:**
```python
def validate(self, value: Any, metadata: Dict) -> ValidationResult:
    # Validate full value
    pass
```

**validate_each() - For lists:**
```python
def validate_each(self, value: Any, metadata: Dict) -> ValidationResult:
    # Called for each element in a list
    pass
```

**validate_stream() - For streaming:**
```python
def validate_stream(
    self, 
    chunk: Any, 
    metadata: Dict, 
    **kwargs
) -> ValidationResult:
    # Validate streaming chunks
    pass
```

**async versions:**
```python
async def validate_async(self, value: Any, metadata: Dict) -> ValidationResult:
    # Async validation
    pass
```

### Validator Configuration

```python
class MyValidator(Validator):
    # Class attributes
    rail_alias = "my-validator"           # Name in RAIL format
    run_in_separate_process = False       # Isolate execution
    override_value_on_pass = False        # Can modify value
    
    # Instance configuration
    def __init__(self, param: str, on_fail: OnFailAction = None):
        super().__init__(on_fail=on_fail)
        self.param = param
    
    def get_args(self) -> Dict:
        """Return configuration for serialization."""
        return {"param": self.param}
```

### Validation Results

**Success:**
```python
from guardrails.classes import PassResult

# Simple pass
return PassResult()

# Pass with modified value
return PassResult(value_override=new_value)

# Pass with metadata
return PassResult(metadata={"info": "..."})
```

**Failure:**
```python
from guardrails.classes import FailResult

# Simple failure
return FailResult(
    error_message="Validation failed"
)

# Failure with suggested fix
return FailResult(
    error_message="Invalid format",
    fix_value="corrected_value"
)

# Failure with error spans (for highlighting)
return FailResult(
    error_message="Contains toxic language",
    error_spans=[
        ErrorSpan(start=0, end=10, reason="Toxic word")
    ]
)
```

## 🎬 OnFailAction API

### Usage

```python
from guardrails import OnFailAction

validator = MyValidator(
    on_fail=OnFailAction.REASK  # or EXCEPTION, FIX, etc.
)
```

### Available Actions

```python
class OnFailAction:
    REASK = "reask"           # Ask LLM to correct
    EXCEPTION = "exception"   # Raise error
    FIX = "fix"               # Apply automatic fix
    FILTER = "filter"         # Remove invalid content
    REFRAIN = "refrain"       # Return None, mark as passed
    FIX_REASK = "fix_reask"  # Try fix, then reask if still invalid
    NOOP = "noop"             # Do nothing, continue
    CUSTOM = "custom"         # Custom handler function
```

### Custom Handler

```python
def custom_handler(
    value: Any,
    fail_result: FailResult,
    metadata: Dict
) -> Any:
    """Custom failure handler."""
    # Custom logic
    return corrected_value

validator = MyValidator(
    on_fail=OnFailAction.CUSTOM,
    on_fail_handler=custom_handler
)
```

## 📊 ValidationOutcome API

### Structure

```python
class ValidationOutcome:
    # Core fields
    raw_llm_output: Optional[str]        # Original LLM response
    validated_output: Any                # Validated/corrected output
    validation_passed: bool              # Overall success
    
    # Error information
    error: Optional[str]                 # Error message
    error_spans: List[ErrorSpan]         # Specific error locations
    
    # Reask information
    reask: Optional[ReAsk]               # Reask details
    
    # Call information  
    call_id: Optional[str]               # Unique call identifier
    
    # Metadata
    metadata: Optional[Dict]             # Additional context
```

### Usage

```python
outcome = guard.validate(value)

# Check success
if outcome.validation_passed:
    # Use validated output
    data = outcome.validated_output
else:
    # Handle errors
    print(f"Error: {outcome.error}")
    for span in outcome.error_spans:
        print(f"  {span.reason} at {span.start}-{span.end}")

# Access raw output (if LLM was called)
if outcome.raw_llm_output:
    print(f"Raw: {outcome.raw_llm_output}")
```

## ⚙️ Settings API

### Configuration

```python
from guardrails import settings

# Hub configuration
settings.hub_api_key = "your-key"
settings.hub_endpoint = "https://api.guardrailsai.com"

# Telemetry
settings.enable_metrics = True
settings.enable_metrics_endpoint = "https://metrics.guardrailsai.com"

# Server mode
settings.use_server = True
settings.api_base_url = "http://localhost:8000"

# Logging
settings.log_level = "INFO"

# Remote inference
settings.enable_remote_inferencing = True
```

### Environment Variables

Can also be set via environment:

```bash
export GUARDRAILS_HUB_API_KEY="your-key"
export GUARDRAILS_LOG_LEVEL="DEBUG"
export GUARDRAILS_USE_SERVER="true"
```

## 📦 Hub API

### Installation

```python
from guardrails import install

# Install validator from Hub
install("hub://guardrails/regex_match")

# Install specific version
install("hub://guardrails/regex_match@1.2.3")
```

**Via CLI:**
```bash
guardrails hub install hub://guardrails/regex_match
```

### Usage

```python
# Import from hub
from guardrails.hub import RegexMatch, ToxicLanguage

# Use in Guard
guard = Guard().use_many(
    RegexMatch(regex=r"\d+"),
    ToxicLanguage(threshold=0.5)
)
```

## 🔌 LLM Integration API

### Prompt Templates

```python
from guardrails import Prompt

prompt = Prompt(
    """
    Generate a person with:
    - Name: A realistic name
    - Age: Between 0 and 150
    
    ${gr.complete_json_suffix_v2}
    """
)

guard = Guard.for_pydantic(
    output_class=Person,
    prompt=prompt
)
```

**Template Variables:**
```python
outcome = guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate {object_type}",
    prompt_params={"object_type": "person"}
)
```

### Messages (Chat Models)

```python
from guardrails import Messages

messages = Messages([
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Generate a person"}
])

guard = Guard.for_pydantic(
    output_class=Person,
    messages=messages
)
```

### Custom LLM Wrapper

```python
from guardrails import PromptCallableBase

class CustomLLM(PromptCallableBase):
    def __call__(self, *args, **kwargs):
        # Custom LLM calling logic
        response = my_llm_api.call(**kwargs)
        return response

# Use with Guard
guard(llm_api=CustomLLM(), prompt="...")
```

## 🔍 Utility Functions

### Validator Utilities

```python
from guardrails.utils.validator_utils import (
    get_validator,              # Get validator class by name
    parse_validator_reference,  # Parse validator reference
)

# Get validator class
ValidatorClass = get_validator("regex_match")

# Instantiate
validator = ValidatorClass(regex=r"\d+")
```

### Parsing Utilities

```python
from guardrails.utils.parsing_utils import (
    parse_llm_output,   # Parse LLM response
    coerce_types,       # Coerce to schema types
)

# Parse output
parsed = parse_llm_output(
    raw_output='{"name": "John"}',
    output_schema=schema,
    output_type=OutputTypes.DICT
)
```

### Prompt Utilities

```python
from guardrails.utils.prompt_utils import (
    prompt_content_for_schema,  # Generate prompt from schema
    messages_to_prompt_string,  # Convert messages to prompt
)

# Generate schema instructions
instructions = prompt_content_for_schema(schema)
```

## 🌐 Server API

### Starting Server

```python
# Via CLI
guardrails start --config=config.py --port=8000

# Via Python
from guardrails_api.cli.start import start
start(env=".env", config="config.py", port=8000)
```

### Client Usage

**Guardrails Client:**
```python
import guardrails as gr

gr.settings.use_server = True

# Use guards from server
guard = gr.Guard(name="my-guard")  # Guard defined on server
outcome = guard.validate(value)
```

**OpenAI-Compatible API:**
```python
import openai

openai.base_url = "http://localhost:8000/guards/my-guard/openai/v1/"

completion = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Generate..."}]
)
```

**Direct HTTP:**
```bash
curl -X POST http://localhost:8000/guards/my-guard/validate \
  -H "Content-Type: application/json" \
  -d '{"value": "test"}'
```

## 📚 Integration APIs

### LangChain

```python
from guardrails.integrations.langchain import GuardrailsRunnable

guard = Guard().use(validator)
runnable = GuardrailsRunnable(guard=guard)

# Use in chain
chain = runnable | other_runnable
result = chain.invoke({"input": "..."})
```

### LlamaIndex

```python
from guardrails.integrations.llama_index import GuardrailsCallbackHandler

guard = Guard().use(validator)
callback = GuardrailsCallbackHandler(guard=guard)

# Use in LlamaIndex
from llama_index import ServiceContext
service_context = ServiceContext.from_defaults(
    callback_manager=callback
)
```

## 🎯 Best Practices

### Error Handling

```python
from guardrails.errors import ValidationError

try:
    outcome = guard.validate(value)
    if not outcome.validation_passed:
        # Handle validation failure
        pass
except ValidationError as e:
    # Handle validation exception (OnFailAction.EXCEPTION)
    print(f"Validation failed: {e}")
except Exception as e:
    # Handle other errors
    print(f"Error: {e}")
```

### Type Hints

```python
from typing import Optional
from guardrails import Guard, ValidationOutcome

def validate_data(data: str) -> Optional[ValidationOutcome]:
    guard = Guard().use(MyValidator())
    return guard.validate(data)
```

### Async Best Practices

```python
import asyncio
from guardrails import AsyncGuard

async def validate_many(values: List[str]):
    guard = AsyncGuard().use(validator)
    
    # Validate concurrently
    tasks = [guard.validate(v) for v in values]
    results = await asyncio.gather(*tasks)
    
    return results
```

## 📖 API Reference Summary

| Component | Module | Description |
|-----------|--------|-------------|
| `Guard` | `guardrails.guard` | Main validation interface |
| `AsyncGuard` | `guardrails.async_guard` | Async validation |
| `Validator` | `guardrails.validator_base` | Base validator class |
| `OnFailAction` | `guardrails.types.on_fail` | Failure handling |
| `ValidationOutcome` | `guardrails.classes.validation_outcome` | Result type |
| `settings` | `guardrails.settings` | Configuration |
| `install` | `guardrails.hub.install` | Hub installation |

## 📚 Next Steps

- [CLI and Server](./05-cli-server.md) - Command-line tools
- [Hub Integration](./06-hub-integration.md) - Validator management
- [Module Reference](./07-module-reference.md) - Implementation details

---

This API structure provides a flexible, extensible interface for validation in AI applications.
