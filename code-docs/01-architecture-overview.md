# Architecture Overview

This document provides a high-level overview of the Guardrails architecture, design patterns, and how different components interact.

## 🏛️ High-Level Architecture

Guardrails follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    User Application Layer                    │
│  (Python code using Guard, validators, LLM integrations)    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      Guard Layer                             │
│  Guard / AsyncGuard - Main entry points for validation      │
│  - Composes validators                                       │
│  - Manages execution flow                                    │
│  - Handles retries and reasks                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     Runner Layer                             │
│  Runner / AsyncRunner / StreamRunner                         │
│  - Orchestrates LLM calls                                    │
│  - Manages validation lifecycle                              │
│  - Implements retry logic                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────┬──────────────────────┬───────────────┐
│   Validator Layer    │   Schema Layer       │  LLM Layer    │
│  - Base Validator    │  - JSON Schema       │  - OpenAI     │
│  - Hub Validators    │  - Pydantic          │  - Anthropic  │
│  - Custom Validators │  - RAIL format       │  - Local LLMs │
└──────────────────────┴──────────────────────┴───────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                       │
│  - Telemetry (OpenTelemetry)                                │
│  - Storage (document_store, vectordb)                       │
│  - API Client (for server mode)                             │
│  - Hub integration (validator management)                    │
└─────────────────────────────────────────────────────────────┘
```

## 🎯 Core Design Patterns

### 1. **Guard Pattern** (Facade + Composition)

The `Guard` class acts as a facade that:
- Composes multiple validators
- Manages execution context
- Provides a simple API for complex validation workflows

```python
# Guards compose validators
guard = Guard().use_many(
    Validator1(...),
    Validator2(...),
    Validator3(...)
)
```

### 2. **Validator Pattern** (Strategy)

Validators implement a common interface (`Validator` base class) allowing:
- Interchangeable validation strategies
- Easy extension with custom validators
- Runtime composition of validation logic

### 3. **Runner Pattern** (Template Method)

The `Runner` class defines the template for:
- LLM invocation
- Output parsing
- Validation execution
- Reask handling

Subclasses (`AsyncRunner`, `StreamRunner`) override specific steps while maintaining the overall workflow.

### 4. **Schema Pattern** (Adapter)

Multiple schema formats are supported through adapters:
- `PydanticSchema` - Converts Pydantic models to internal schema
- `RailSchema` - Converts RAIL format to internal schema
- `JSONSchema` - Works directly with JSON Schema

### 5. **Action Pattern** (Command)

Validation failures trigger actions:
- `Reask` - Request corrected output from LLM
- `Refrain` - Skip validation and continue
- `Filter` - Remove invalid content
- `Exception` - Raise an error
- `Fix` - Attempt automatic correction

## 🔄 Data Flow

### Basic Validation Flow

```
User Input → Guard → Validator(s) → Result
                ↓                      ↑
              LLM (optional)     Validated Output
```

### LLM Call with Validation Flow

```
1. User provides prompt + Guard configuration
                ↓
2. Guard prepares schema and validators
                ↓
3. Runner calls LLM API
                ↓
4. Parse LLM response
                ↓
5. Run validators on output
                ↓
6. Check validation results
   ├─ Success → Return validated output
   └─ Failure → Execute OnFailAction
                  ├─ Reask → Modify prompt, go to step 3
                  ├─ Fix → Attempt repair, go to step 5
                  ├─ Filter → Remove invalid parts, continue
                  └─ Exception → Raise error
```

## 🧩 Key Components

### Guard (`guard.py`, `async_guard.py`)

**Responsibilities:**
- Main entry point for users
- Validator composition and management
- Schema management (Pydantic, JSON Schema, RAIL)
- Execution orchestration
- Context management

**Key Methods:**
- `use()` / `use_many()` - Add validators
- `validate()` - Validate raw input/output
- `__call__()` - Validate with LLM integration
- `parse()` - Parse and validate structured data

### Validator (`validator_base.py`)

**Responsibilities:**
- Define validation logic
- Handle different validation scopes (full, element, sentence)
- Support sync and async execution
- Integrate with remote inference (for ML-based validators)

**Key Methods:**
- `validate()` - Main validation entry point
- `validate_stream()` - Streaming validation
- `to_dict()` - Serialization for storage/transmission

### Runner (`run/runner.py`, `run/async_runner.py`)

**Responsibilities:**
- Orchestrate LLM API calls
- Manage prompt construction
- Handle response parsing
- Execute validation chain
- Implement retry/reask logic
- Collect telemetry

**Key Methods:**
- `step()` - Execute one validation iteration
- `__call__()` - Run complete validation workflow

### Schema System (`schema/`)

**Responsibilities:**
- Convert various schema formats to internal representation
- Generate prompts from schemas
- Validate data against schemas
- Support structured output generation

**Key Modules:**
- `pydantic_schema.py` - Pydantic model conversion
- `rail_schema.py` - RAIL format parsing
- `json_schema.py` - JSON Schema handling
- `validator.py` - Schema validation logic

## 🌐 Execution Modes

### 1. **Standalone Mode**

Direct Python usage with Guards and validators:

```python
from guardrails import Guard
guard = Guard().use(SomeValidator())
result = guard.validate(data)
```

### 2. **LLM Integration Mode**

Guards validate LLM inputs/outputs:

```python
guard = Guard().use(SomeValidator())
result = guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate text...",
    model="gpt-4"
)
```

### 3. **Server Mode**

Guards deployed as REST API:

```bash
guardrails start --config=config.py --port=8000
```

Accessible via:
- Guardrails client
- OpenAI-compatible API
- Direct HTTP requests

### 4. **Integration Mode**

Embedded in other frameworks:
- LangChain integration (`integrations/langchain/`)
- LlamaIndex integration (`integrations/llama_index/`)
- Custom integrations via `PromptCallableBase`

## 📊 State Management

### Context Variables

Guardrails uses Python's `contextvars` for thread-safe state:

```python
# stores/context.py
tracer_context = ContextVar("tracer_context")
guard_context = ContextVar("guard_context")
```

This enables:
- Per-request isolation
- Async-safe execution
- Distributed tracing support

### History Tracking

Every Guard execution creates a `Call` object:

```python
class Call:
    inputs: CallInputs
    iterations: List[Iteration]
    outputs: Outputs
```

This provides:
- Full execution history
- Debugging information
- Audit trail

## 🔌 Extension Points

### Custom Validators

Extend `Validator` base class:

```python
from guardrails import Validator, register_validator

@register_validator(name="my_validator", data_type="string")
class MyValidator(Validator):
    def validate(self, value, metadata):
        # Custom validation logic
        pass
```

### Custom LLM Providers

Implement `PromptCallableBase`:

```python
from guardrails import PromptCallableBase

class MyLLMProvider(PromptCallableBase):
    def __call__(self, *args, **kwargs):
        # Custom LLM calling logic
        pass
```

### Custom Actions

Implement custom failure handling:

```python
from guardrails.actions import ActionBase

class CustomAction(ActionBase):
    def apply(self, value, metadata):
        # Custom failure handling
        pass
```

## 🔐 Security Considerations

### Input Validation

- All user inputs are validated before LLM calls
- Prevents prompt injection when properly configured
- Schema validation ensures type safety

### Output Validation

- LLM outputs are validated before returning to user
- Prevents data leakage through validation
- Customizable validation strictness

### Hub Validators

- Downloaded from trusted Hub
- JWT-based authentication
- Package verification

## 🚀 Performance Optimization

### Caching

- Schema compilation is cached
- Validator instances are reused
- LLM responses can be cached (user-configurable)

### Async Support

- Full async/await throughout the stack
- Non-blocking I/O for network operations
- Parallel validator execution where possible

### Streaming

- Support for streaming LLM responses
- Incremental validation during streaming
- Reduced time-to-first-token perception

## 📈 Observability

### Telemetry

- OpenTelemetry integration
- Spans for each major operation
- Custom attributes for debugging

### Logging

- Structured logging with `rich`
- Configurable log levels
- Context-aware log messages

### Metrics

- Validation success/failure rates
- LLM call latency
- Reask counts
- Hub telemetry (opt-in)

## 🔄 Versioning Strategy

- **Semantic Versioning**: Major.Minor.Patch
- **Deprecation Warnings**: Features marked deprecated before removal
- **Backward Compatibility**: Maintained within major versions
- **Schema Evolution**: Validators maintain compatibility contracts

## 🎓 Learning Path

For new developers:

1. **Start Here**: `guard.py` - Understand the main Guard class
2. **Then**: `validator_base.py` - See how validators work
3. **Next**: `run/runner.py` - Understand execution flow
4. **Finally**: Specific modules based on your interest

## 📚 Related Documentation

- [Core Concepts](./02-core-concepts.md) - Deep dive into Guards and Validators
- [Workflow and Execution](./03-workflow-execution.md) - Detailed execution flow
- [Module Reference](./07-module-reference.md) - Individual module documentation

---

This architecture supports the core mission of Guardrails: making AI applications reliable through composable, reusable validation logic.
