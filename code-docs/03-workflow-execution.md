# Workflow and Execution

This document explains how Guardrails processes requests from start to finish, detailing the complete execution flow.

## 🔄 Complete Execution Flow

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User creates Guard with validators                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. User calls Guard with input/prompt                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Guard prepares Runner with configuration                 │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Runner calls LLM (if needed)                            │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Parse LLM response                                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Execute validators                                        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. Process validation results                               │
│    ├─ All pass → Return output                             │
│    └─ Some fail → Handle OnFailAction                      │
│        ├─ Reask → Go back to step 4                        │
│        ├─ Fix → Apply fix, go to step 6                    │
│        ├─ Filter → Remove invalid, continue                │
│        └─ Exception/Refrain → Process and return           │
└─────────────────────────────────────────────────────────────┘
```

## 📋 Detailed Step-by-Step Flow

### Step 1: Guard Creation

**Location:** `guardrails/guard.py`

```python
# User creates a Guard
guard = Guard().use_many(
    Validator1(...),
    Validator2(...)
)
```

**What happens:**
1. Guard instance is created
2. Validators are added to internal list
3. Validator map is built (maps schema paths to validators)
4. Schema is prepared (if provided)

**Code path:**
```
Guard.__init__()
  → Guard.configure()
  → Guard.use_many()
    → Guard._add_validator()
      → build validator_map
```

### Step 2: Guard Invocation

**Location:** `guardrails/guard.py:Guard.__call__()`

```python
result = guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate a person's info",
    model="gpt-4"
)
```

**What happens:**
1. Input validation runs (if input validators exist)
2. Context is prepared (tracer, metadata)
3. Prompt is constructed with schema
4. Runner is created
5. Execution begins

**Code path:**
```
Guard.__call__()
  → Guard._execute()
    → Guard._call()
      → create Runner
      → Runner.__call__()
```

### Step 3: Runner Initialization

**Location:** `guardrails/run/runner.py`

```python
runner = Runner(
    output_type=OutputTypes.DICT,
    output_schema=schema,
    num_reasks=2,
    validation_map=validator_map,
    api=llm_api,
    messages=messages
)
```

**What happens:**
1. Runner stores configuration
2. Prepares prompt with schema instructions
3. Sets up validation chain
4. Initializes iteration counter
5. Creates Call object for history

**Code path:**
```
Runner.__init__()
  → store configuration
  → prepare validation_map
```

### Step 4: LLM API Call

**Location:** `guardrails/run/runner.py:Runner.step()`

**First Iteration:**
```python
# Runner calls LLM with prepared prompt
response = self.api(
    messages=messages,
    model=model,
    **kwargs
)
```

**What happens:**
1. Prompt is formatted with schema
2. LLM API is called
3. Response is received
4. Raw output is stored
5. Telemetry is recorded

**Code path:**
```
Runner.step()
  → Runner._call_llm()
    → llm_api(**kwargs)
    → LLMResponse object created
```

**Subsequent Iterations (Reask):**
```python
# On validation failure with REASK
# New prompt includes error information
reask_prompt = f"""
Previous output was invalid:
{error_message}

Please correct:
{original_prompt}
"""
```

### Step 5: Response Parsing

**Location:** `guardrails/utils/parsing_utils.py`

```python
parsed_output = parse_llm_output(
    raw_output,
    output_schema,
    output_type
)
```

**What happens:**
1. Extract content from LLM response
2. Parse based on output_type:
   - **JSON**: Parse and validate structure
   - **String**: Return as-is
   - **List**: Parse array
3. Coerce types to match schema
4. Handle parsing errors

**Code path:**
```
parse_llm_output()
  → extract_json() or extract_text()
  → coerce_types()
  → prune_extra_keys()
```

**Error Handling:**
```python
try:
    parsed = json.loads(raw_output)
except JSONDecodeError:
    # Create NonParseableReAsk
    return NonParseableReAsk(...)
```

### Step 6: Validator Execution

**Location:** `guardrails/run/runner.py` + `guardrails/validator_service/`

**Sequential Execution:**

```python
# For each validator in the validation_map
for path, validators in validation_map.items():
    value = get_value_at_path(parsed_output, path)
    
    for validator in validators:
        result = validator.validate(value, metadata)
        
        if isinstance(result, FailResult):
            # Handle failure
            handle_validation_failure(result, validator)
```

**What happens:**
1. Validators run in order on their target paths
2. Each validator returns PassResult or FailResult
3. Results are collected
4. Metadata is updated

**Code path:**
```
Runner.step()
  → validate_dependents()
    → validator_service.validate()
      → Validator.validate()
        → PassResult or FailResult
```

**Validation Scopes:**

Different validators operate at different scopes:

**Full Value:**
```python
def validate(self, value, metadata):
    # Validates entire value
    return PassResult() or FailResult()
```

**Each Element:**
```python
def validate_each(self, value, metadata):
    # Called for each item in a list
    results = []
    for item in value:
        result = self.validate_item(item)
        results.append(result)
    return results
```

**Streaming:**
```python
def validate_stream(self, chunk, metadata):
    # Validates chunks as they arrive
    return PassResult() or FailResult()
```

### Step 7: Result Processing

**Location:** `guardrails/run/runner.py`

**All Validators Pass:**

```python
if all_passed:
    return ValidationOutcome(
        raw_llm_output=raw_output,
        validated_output=validated_output,
        validation_passed=True
    )
```

**Some Validators Fail:**

Process based on `OnFailAction`:

#### **REASK Action:**

```python
if on_fail == OnFailAction.REASK:
    if iteration < num_reasks:
        # Create reask prompt
        reask = ReAsk(
            incorrect_value=value,
            error_message=error_message,
            fix_value=None
        )
        
        # Add to next iteration
        new_prompt = construct_reask_prompt(
            original_prompt,
            reask
        )
        
        # Go back to step 4 (LLM call)
        return runner.step(iteration + 1)
    else:
        # Max reasks reached
        return final_result_with_errors()
```

**Code path:**
```
Runner.step()
  → validation fails
  → get_reask_setup()
  → construct reask prompt
  → Runner.step(iteration + 1)
```

#### **FIX Action:**

```python
if on_fail == OnFailAction.FIX:
    if fail_result.fix_value is not None:
        # Apply the fix
        value = fail_result.fix_value
        
        # Re-run validation with fixed value
        return runner.step(
            iteration,
            output=fixed_value
        )
    else:
        # No fix available, return error
        return error_outcome()
```

#### **FILTER Action:**

```python
if on_fail == OnFailAction.FILTER:
    # Remove invalid elements
    if isinstance(value, list):
        value = [
            item for item in value
            if validate_item(item).passed
        ]
    else:
        value = None
    
    # Continue with filtered value
    return ValidationOutcome(
        validated_output=value,
        validation_passed=True  # Treated as pass
    )
```

#### **REFRAIN Action:**

```python
if on_fail == OnFailAction.REFRAIN:
    # Return None/empty, mark as passed
    return ValidationOutcome(
        validated_output=None,
        validation_passed=True
    )
```

#### **EXCEPTION Action:**

```python
if on_fail == OnFailAction.EXCEPTION:
    raise ValidationError(
        error_message,
        error_spans=error_spans
    )
```

## 🔀 Execution Variants

### Simple Validation (No LLM)

```python
guard = Guard().use(SomeValidator())
result = guard.validate("some value")
```

**Flow:**
1. Skip LLM call
2. Go directly to validation
3. Return result

**Code path:**
```
Guard.validate()
  → validate_dependents()
  → validator.validate()
  → return ValidationOutcome
```

### Parse and Validate

```python
result = guard.parse(
    llm_output='{"name": "John", "age": 30}',
    metadata={"request_id": "123"}
)
```

**Flow:**
1. Parse provided output
2. Run validators
3. Return result

**Code path:**
```
Guard.parse()
  → parse_llm_output()
  → validate_dependents()
  → return ValidationOutcome
```

### Streaming Execution

```python
for fragment in guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate...",
    stream=True
):
    print(fragment.validated_output)
```

**Flow:**
1. LLM returns chunks
2. Accumulate chunks
3. Validate incrementally (if supported)
4. Yield validated fragments

**Code path:**
```
Guard.__call__(stream=True)
  → StreamRunner.__call__()
    → for chunk in llm_stream:
      → accumulate_chunk()
      → validate_stream()
      → yield Fragment
```

**Location:** `guardrails/run/stream_runner.py`

### Async Execution

```python
result = await async_guard(
    llm_api=openai.chat.completions.create,
    prompt="Generate..."
)
```

**Flow:**
1. Async LLM call
2. Async validation
3. Return result

**Code path:**
```
AsyncGuard.__call__()
  → AsyncRunner.__call__()
    → await llm_api()
    → await validate_async()
    → return ValidationOutcome
```

**Location:** `guardrails/async_guard.py`, `guardrails/run/async_runner.py`

## 🔄 Reask Flow Deep Dive

### Reask Context

When validation fails with REASK:

```python
# Original call
guard(prompt="Generate a person")
→ {"name": "John", "age": "thirty"}  # Invalid age

# Reask is triggered
reask_prompt = """
Previous output had errors:
- age must be an integer, got "thirty"

Please provide corrected output:
Generate a person
"""

# Second LLM call
→ {"name": "John", "age": 30}  # Valid!
```

### Reask Prompt Construction

**Location:** `guardrails/actions/reask.py`

```python
def get_reask_setup(
    reasks: List[ReAsk],
    original_prompt: str,
    original_messages: List[Dict]
) -> Tuple[str, List[Dict]]:
    """
    Constructs reask prompt with error information.
    """
    error_messages = [
        f"- {reask.error_message}"
        for reask in reasks
    ]
    
    reask_prompt = f"""
    {original_prompt}
    
    Previous output had the following errors:
    {chr(10).join(error_messages)}
    
    Please provide corrected output.
    """
    
    return reask_prompt, updated_messages
```

### Iteration Tracking

Each iteration is stored:

```python
class Call:
    iterations: List[Iteration]

# After each step
iteration = Iteration(
    inputs=inputs,
    outputs=outputs,
    validator_results=results,
    iteration_number=i
)
call.iterations.append(iteration)
```

## 📊 State Management

### Context Variables

**Location:** `guardrails/stores/context.py`

```python
# Thread-safe context
tracer_context = ContextVar("tracer_context")
guard_context = ContextVar("guard_context")

# Set context for request
set_tracer_context(tracer)
set_guard_name(guard.name)

# Access later
tracer = get_tracer_context()
```

### Call History

Every Guard execution creates history:

```python
call = Call(
    inputs=CallInputs(...),
    iterations=[],
    outputs=Outputs(...),
    validator_logs=[],
    start_time=time.time(),
    end_time=None
)

# After execution
guard.history.append(call)
```

Access history:

```python
# Get last call
last_call = guard.history.last

# Get all calls
all_calls = guard.history

# Access iterations
for iteration in last_call.iterations:
    print(f"Iteration {iteration.number}")
    print(f"Output: {iteration.outputs}")
```

## 🎯 Validation Execution Details

### Validator Service

**Location:** `guardrails/validator_service/`

Central validation orchestration:

```python
def validate(
    value: Any,
    validators: List[Validator],
    metadata: Dict[str, Any],
    **kwargs
) -> List[ValidationResult]:
    """
    Runs validators on a value.
    """
    results = []
    
    for validator in validators:
        try:
            result = validator.validate(value, metadata)
            results.append(result)
        except Exception as e:
            results.append(FailResult(
                error_message=str(e)
            ))
    
    return results
```

### Parallel Validation

Some validators can run in parallel:

```python
async def validate_parallel(
    value: Any,
    validators: List[Validator],
    metadata: Dict[str, Any]
) -> List[ValidationResult]:
    """
    Run independent validators in parallel.
    """
    tasks = [
        validator.validate_async(value, metadata)
        for validator in validators
    ]
    
    results = await asyncio.gather(*tasks)
    return results
```

## 🔍 Error Handling

### Error Types

**Validation Errors:**
```python
class ValidationError(Exception):
    """Raised when validation fails with EXCEPTION."""
    pass
```

**Parsing Errors:**
```python
class NonParseableReAsk:
    """Indicates LLM output couldn't be parsed."""
    incorrect_value: str
    error_message: str
```

**User Errors:**
```python
class UserFacingException(Exception):
    """Errors shown to end users."""
    pass
```

### Error Recovery

Guards handle errors gracefully:

```python
try:
    result = guard(...)
except ValidationError as e:
    # Validation failed with EXCEPTION
    print(f"Validation error: {e}")
except Exception as e:
    # Other errors (API, network, etc.)
    print(f"Error: {e}")
```

## 📈 Performance Considerations

### Caching

Schema compilation is cached:

```python
@lru_cache(maxsize=128)
def compile_schema(schema_dict):
    """Cache compiled schemas."""
    return ProcessedSchema(schema_dict)
```

### Lazy Loading

Validators are loaded on-demand:

```python
def get_validator(name: str) -> Type[Validator]:
    """Load validator class lazily."""
    if name not in _validator_cache:
        _validator_cache[name] = import_validator(name)
    return _validator_cache[name]
```

### Streaming Benefits

Streaming reduces latency:

```
Traditional:
  Wait for full LLM response (10s) → Validate → Return
  Total: 10s + validation time

Streaming:
  Receive chunks (0.1s each) → Validate → Yield
  First output: 0.1s + validation time
```

## 🎓 Key Takeaways

1. **Execution is orchestrated by Runner** - centralized workflow
2. **Validation happens after parsing** - ensures valid structure first
3. **Reask creates new iterations** - with error context
4. **State is tracked in Call/Iteration** - full history available
5. **Context variables enable isolation** - thread-safe execution
6. **Streaming enables incremental validation** - better UX
7. **Error handling is multi-layered** - graceful degradation

## 📚 Next Steps

- [API Structure](./04-api-structure.md) - Learn public APIs
- [Module Reference](./07-module-reference.md) - Dive into implementations
- [Development Guide](./08-development-guide.md) - Contributing workflow

---

Understanding this execution flow is essential for debugging, optimization, and extending Guardrails.
