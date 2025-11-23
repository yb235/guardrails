# Module Reference

This document provides a detailed breakdown of key modules and their responsibilities in the Guardrails codebase.

## 📁 Module Organization

```
guardrails/
├── guard.py                  # Main Guard class
├── async_guard.py            # Async Guard implementation
├── validator_base.py         # Base Validator class
├── llm_providers.py          # LLM integration interfaces
├── settings.py               # Configuration management
├── actions/                  # Failure handling actions
├── classes/                  # Core data structures
├── cli/                      # Command-line interface
├── hub/                      # Hub integration
├── run/                      # Execution runners
├── schema/                   # Schema processing
├── validators/               # Built-in validators
├── utils/                    # Utility functions
├── telemetry/                # Observability
└── ...                       # Other modules
```

## 🛡️ Core Modules

### guard.py

**Purpose:** Main Guard class implementation

**Key Classes:**
- `Guard` - Primary validation interface

**Responsibilities:**
- Validator composition
- Schema management
- LLM integration orchestration
- History tracking
- Execution delegation

**Key Methods:**
```python
class Guard:
    # Creation methods
    @classmethod
    def for_pydantic(cls, output_class, ...)
    @classmethod
    def from_dict(cls, schema, ...)
    @classmethod
    def from_rail(cls, rail_file, ...)
    
    # Validator management
    def use(self, validator_class, **kwargs)
    def use_many(self, *validators)
    
    # Validation methods
    def validate(self, value, metadata=None, ...)
    def parse(self, llm_output, ...)
    def __call__(self, llm_api=None, ...)
    
    # Serialization
    def to_dict(self)
    def to_config(self)
```

**Dependencies:**
- `validator_base.py` - Validator interface
- `run/runner.py` - Execution orchestration
- `schema/` - Schema processing
- `classes/` - Data structures

**Usage Example:**
```python
from guardrails import Guard

guard = Guard().use(MyValidator())
result = guard.validate("data")
```

---

### async_guard.py

**Purpose:** Asynchronous Guard implementation

**Key Classes:**
- `AsyncGuard` - Async version of Guard

**Differences from Guard:**
- Uses `AsyncRunner` instead of `Runner`
- All methods are async/await
- Supports async validators
- Concurrent validation

**Key Methods:**
```python
class AsyncGuard:
    async def validate(self, value, ...)
    async def parse(self, llm_output, ...)
    async def __call__(self, llm_api=None, ...)
```

**Usage Example:**
```python
from guardrails import AsyncGuard

guard = AsyncGuard().use(MyAsyncValidator())
result = await guard.validate("data")
```

---

### validator_base.py

**Purpose:** Base class for all validators

**Key Classes:**
- `Validator` - Abstract base validator

**Responsibilities:**
- Define validator interface
- Handle validation scopes
- Support streaming
- Manage configuration

**Key Attributes:**
```python
class Validator:
    rail_alias: str                      # Name in RAIL format
    run_in_separate_process: bool        # Process isolation
    override_value_on_pass: bool         # Can modify values
```

**Key Methods:**
```python
def validate(self, value, metadata) -> ValidationResult
def validate_each(self, value, metadata) -> ValidationResult
def validate_stream(self, chunk, metadata) -> ValidationResult
async def validate_async(self, value, metadata) -> ValidationResult

def to_dict(self) -> dict
def get_args(self) -> dict
```

**Decorator:**
```python
@register_validator(name="my_validator", data_type="string")
class MyValidator(Validator):
    pass
```

---

### llm_providers.py

**Purpose:** LLM integration interfaces

**Key Classes:**
- `PromptCallableBase` - Base for LLM wrappers
- `AsyncPromptCallableBase` - Async version

**Responsibilities:**
- Define LLM API interface
- Standardize LLM responses
- Support multiple providers

**Key Functions:**
```python
def get_llm_api_enum(llm_api) -> str
def get_llm_ask(llm_api, **kwargs)
def model_is_supported_server_side(model: str) -> bool
```

**Usage:**
```python
class CustomLLM(PromptCallableBase):
    def __call__(self, *args, **kwargs):
        # LLM logic
        return response
```

---

### settings.py

**Purpose:** Global configuration management

**Key Class:**
- `Settings` - Configuration singleton

**Configuration Options:**
```python
class Settings:
    # Hub
    hub_api_key: Optional[str]
    hub_endpoint: str
    
    # Telemetry
    enable_metrics: bool
    enable_metrics_endpoint: str
    
    # Server
    use_server: bool
    api_base_url: str
    
    # Logging
    log_level: str
    
    # Remote inference
    enable_remote_inferencing: bool
```

**Usage:**
```python
from guardrails import settings

settings.hub_api_key = "your-key"
settings.use_server = True
```

---

## 🏃 Run Module

### run/runner.py

**Purpose:** Synchronous execution orchestration

**Key Class:**
- `Runner` - Orchestrates validation workflow

**Responsibilities:**
- LLM API calls
- Response parsing
- Validator execution
- Reask handling
- History tracking

**Workflow:**
```python
def step(self, iteration: int) -> ValidationOutcome:
    # 1. Call LLM
    llm_response = self.call_llm()
    
    # 2. Parse response
    parsed = parse_llm_output(llm_response)
    
    # 3. Run validators
    validation_results = validate_dependents(parsed)
    
    # 4. Handle results
    if all_passed:
        return success_outcome(parsed)
    else:
        return handle_failures(validation_results)
```

---

### run/async_runner.py

**Purpose:** Asynchronous execution

**Key Class:**
- `AsyncRunner` - Async version of Runner

**Differences:**
- Async LLM calls
- Concurrent validator execution
- Async/await throughout

---

### run/stream_runner.py

**Purpose:** Streaming execution

**Key Class:**
- `StreamRunner` - Handles streaming LLM responses

**Responsibilities:**
- Accumulate chunks
- Incremental validation
- Yield validated fragments

**Workflow:**
```python
def __call__(self):
    for chunk in llm_stream:
        # Accumulate
        self.accumulator += chunk
        
        # Validate if possible
        if can_validate(self.accumulator):
            result = validate_chunk(self.accumulator)
            yield result
```

---

## 🏗️ Classes Module

### classes/validation/

**Purpose:** Validation result types

**Key Classes:**
```python
# Result types
class PassResult:
    value_override: Optional[Any]
    metadata: Dict

class FailResult:
    error_message: str
    fix_value: Optional[Any]
    error_spans: List[ErrorSpan]

# Error locations
class ErrorSpan:
    start: int
    end: int
    reason: str
```

---

### classes/history/

**Purpose:** Execution history tracking

**Key Classes:**
```python
class Call:
    """Complete Guard execution."""
    inputs: CallInputs
    iterations: List[Iteration]
    outputs: Outputs
    validator_logs: List[ValidatorLog]

class Iteration:
    """Single validation attempt."""
    call_id: str
    index: int
    inputs: Inputs
    outputs: Outputs
```

**Usage:**
```python
guard = Guard().use(validator)
result = guard.validate(value)

# Access history
last_call = guard.history.last
for iteration in last_call.iterations:
    print(iteration.outputs)
```

---

### classes/validation_outcome.py

**Purpose:** Validation result wrapper

**Key Class:**
```python
class ValidationOutcome:
    raw_llm_output: Optional[str]
    validated_output: Any
    validation_passed: bool
    error: Optional[str]
    error_spans: List[ErrorSpan]
    reask: Optional[ReAsk]
```

---

## 📋 Schema Module

### schema/pydantic_schema.py

**Purpose:** Convert Pydantic models to internal schema

**Key Functions:**
```python
def pydantic_model_to_schema(
    model: Type[BaseModel]
) -> Dict[str, Any]:
    """Convert Pydantic model to JSON Schema."""
    return model.model_json_schema()
```

---

### schema/rail_schema.py

**Purpose:** Parse RAIL format

**Key Functions:**
```python
def rail_file_to_schema(rail_file: str) -> Dict:
    """Parse RAIL file to schema."""
    with open(rail_file) as f:
        rail_string = f.read()
    return rail_string_to_schema(rail_string)

def rail_string_to_schema(rail_string: str) -> Dict:
    """Parse RAIL string to schema."""
    # Parse XML
    # Extract output schema
    # Convert to JSON Schema
    return schema
```

---

### schema/validator.py

**Purpose:** Schema validation

**Key Functions:**
```python
def validate_json_schema(schema: Dict):
    """Validate schema is correct."""
    # Check required fields
    # Verify types
    # Ensure valid structure

def schema_validation(
    data: Any,
    schema: Dict,
    validator_map: ValidatorMap
) -> List[ValidationResult]:
    """Validate data against schema."""
    # Check structure matches schema
    # Run validators on fields
    return results
```

---

## 🎬 Actions Module

### actions/reask.py

**Purpose:** Reask action implementation

**Key Classes:**
```python
class ReAsk:
    """Information for reask."""
    incorrect_value: Any
    error_message: str
    fix_value: Optional[Any]

class NonParseableReAsk(ReAsk):
    """LLM output couldn't be parsed."""
    pass
```

**Key Functions:**
```python
def get_reask_setup(
    reasks: List[ReAsk],
    original_prompt: str,
    original_messages: List[Dict]
) -> Tuple[str, List[Dict]]:
    """Construct reask prompt with errors."""
    # Add error information
    # Request corrections
    return reask_prompt, reask_messages
```

---

### actions/refrain.py

**Purpose:** Refrain action (skip validation)

**Key Class:**
```python
class Refrain:
    """Marker for refrain action."""
    pass
```

---

### actions/filter.py

**Purpose:** Filter action (remove invalid elements)

**Key Functions:**
```python
def filter_invalid_elements(
    value: List[Any],
    validation_results: List[ValidationResult]
) -> List[Any]:
    """Remove elements that failed validation."""
    return [
        v for v, r in zip(value, validation_results)
        if r.passed
    ]
```

---

## 🔌 Hub Module

### hub/install.py

**Purpose:** Install validators from Hub

**Key Functions:**
```python
def install(
    uri: str,
    upgrade: bool = False,
    quiet: bool = False
):
    """Install validator from Hub."""
    # Parse URI
    # Download package
    # Install dependencies
    # Register validator
```

---

### hub/validator_package_service.py

**Purpose:** Manage installed validators

**Key Class:**
```python
class ValidatorPackageService:
    def get_installed_validators(self) -> List[str]
    def get_validator_path(self, name: str) -> Path
    def get_validator_version(self, name: str) -> str
    def is_installed(self, name: str) -> bool
```

---

## 🛠️ Utils Module

### utils/validator_utils.py

**Purpose:** Validator utilities

**Key Functions:**
```python
def get_validator(name: str) -> Type[Validator]:
    """Get validator class by name."""
    # Check registry
    # Import if needed
    return ValidatorClass

def parse_validator_reference(ref: str) -> ValidatorReference:
    """Parse validator reference string."""
    # Parse format: "validator_name: arg1=val1 arg2=val2"
    return ValidatorReference(...)
```

---

### utils/parsing_utils.py

**Purpose:** LLM output parsing

**Key Functions:**
```python
def parse_llm_output(
    raw_output: str,
    output_schema: Dict,
    output_type: OutputTypes
) -> Any:
    """Parse LLM response based on type."""
    if output_type == OutputTypes.DICT:
        return json.loads(raw_output)
    elif output_type == OutputTypes.STRING:
        return raw_output.strip()
    # ... other types

def coerce_types(data: Dict, schema: Dict) -> Dict:
    """Coerce data types to match schema."""
    # Convert strings to ints, etc.
    return coerced_data
```

---

### utils/prompt_utils.py

**Purpose:** Prompt construction

**Key Functions:**
```python
def prompt_content_for_schema(schema: Dict) -> str:
    """Generate prompt instructions from schema."""
    # Describe expected structure
    # Add validation requirements
    return instructions

def messages_to_prompt_string(messages: List[Dict]) -> str:
    """Convert chat messages to single prompt."""
    return "\n".join(msg["content"] for msg in messages)
```

---

## 📡 Telemetry Module

### telemetry/

**Purpose:** Observability with OpenTelemetry

**Key Functions:**
```python
def trace_guard_execution(guard_name: str):
    """Decorator for tracing guard execution."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            with trace_span(f"guard.{guard_name}"):
                return func(*args, **kwargs)
        return wrapper
    return decorator

def trace_call(func):
    """Trace LLM calls."""
    pass

def trace_step(func):
    """Trace validation steps."""
    pass
```

---

## 🗄️ Stores Module

### stores/context.py

**Purpose:** Context management with contextvars

**Key Variables:**
```python
tracer_context: ContextVar[Tracer]
guard_context: ContextVar[Guard]
call_kwargs: ContextVar[Dict]
```

**Key Functions:**
```python
def set_tracer(tracer: Tracer):
    """Set tracer for current context."""
    tracer_context.set(tracer)

def get_tracer_context() -> Tracer:
    """Get current tracer."""
    return tracer_context.get()

def set_guard_name(name: str):
    """Set guard name for context."""
    pass
```

---

## 🔍 Validators Module

### validators/

**Purpose:** Built-in validators

**Validators included:**
- String validators
- Numeric validators  
- Structural validators

**Example:**
```python
# validators/string_validators.py
from guardrails import Validator

@register_validator(name="length", data_type="string")
class Length(Validator):
    def __init__(self, min: int = None, max: int = None, **kwargs):
        super().__init__(**kwargs)
        self.min = min
        self.max = max
    
    def validate(self, value: str, metadata: dict):
        length = len(value)
        if self.min and length < self.min:
            return FailResult(f"Too short: {length} < {self.min}")
        if self.max and length > self.max:
            return FailResult(f"Too long: {length} > {self.max}")
        return PassResult()
```

---

## 🧩 Integration Modules

### integrations/langchain/

**Purpose:** LangChain integration

**Key Classes:**
```python
class GuardrailsRunnable(Runnable):
    """Runnable wrapper for Guards."""
    
    def __init__(self, guard: Guard):
        self.guard = guard
    
    def invoke(self, input: dict) -> dict:
        result = self.guard.validate(input["value"])
        return {"output": result.validated_output}
```

---

### integrations/llama_index/

**Purpose:** LlamaIndex integration

**Key Classes:**
```python
class GuardrailsCallbackHandler:
    """Callback for LlamaIndex."""
    
    def __init__(self, guard: Guard):
        self.guard = guard
    
    def on_llm_end(self, response):
        # Validate LLM response
        self.guard.validate(response)
```

---

## 📚 Module Dependencies

### Dependency Graph

```
guard.py
├─→ validator_base.py
├─→ run/runner.py
│   ├─→ llm_providers.py
│   ├─→ utils/parsing_utils.py
│   └─→ validator_service/
├─→ schema/
│   ├─→ pydantic_schema.py
│   ├─→ rail_schema.py
│   └─→ validator.py
└─→ classes/
    ├─→ validation/
    ├─→ history/
    └─→ validation_outcome.py
```

## 🎯 Module Design Patterns

### 1. Separation of Concerns
- Guards handle orchestration
- Runners handle execution
- Validators handle validation logic
- Schemas handle structure

### 2. Dependency Injection
- LLM APIs injected into Guards
- Validators injected into Guards
- Configuration injected via settings

### 3. Factory Pattern
- `Guard.for_pydantic()`
- `Guard.from_dict()`
- `Guard.from_rail()`

### 4. Strategy Pattern
- Validators are interchangeable strategies
- OnFailActions are pluggable strategies

### 5. Observer Pattern
- Telemetry observes execution
- History tracks all operations

## 📖 Quick Reference

| Module | Purpose | Key Exports |
|--------|---------|-------------|
| `guard.py` | Main interface | `Guard` |
| `async_guard.py` | Async interface | `AsyncGuard` |
| `validator_base.py` | Validator base | `Validator`, `register_validator` |
| `llm_providers.py` | LLM integration | `PromptCallableBase` |
| `run/runner.py` | Execution | `Runner` |
| `actions/` | Failure handling | `ReAsk`, `Refrain`, `Filter` |
| `classes/` | Data structures | `ValidationOutcome`, `Call` |
| `schema/` | Schema processing | `pydantic_model_to_schema` |
| `hub/` | Hub integration | `install` |
| `utils/` | Utilities | Various helpers |

## 📚 Next Steps

- [Development Guide](./08-development-guide.md) - Contributing to modules
- [Testing Infrastructure](./09-testing-infrastructure.md) - Testing modules
- [API Structure](./04-api-structure.md) - Public API reference

---

Understanding the module structure helps navigate and contribute to the codebase effectively.
