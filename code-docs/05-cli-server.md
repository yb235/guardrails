# CLI and Server

This document covers the command-line interface (CLI) and server deployment capabilities of Guardrails.

## 🖥️ CLI Overview

The Guardrails CLI is implemented using Typer and provides commands for managing Guards, validators, and server deployment.

**Location:** `guardrails/cli/`

**Entry Point:** `guardrails` command (defined in `pyproject.toml`)

```toml
[project.scripts]
guardrails = "guardrails.cli:cli"
```

## 📋 CLI Commands

### configure

Configure Guardrails settings (Hub API key, endpoints, etc.)

```bash
guardrails configure
```

**Interactive prompts for:**
- Hub API token
- Enable/disable telemetry
- Remote inferencing settings

**Implementation:** `guardrails/cli/configure.py`

**What it does:**
1. Prompts for configuration values
2. Stores in `~/.guardrailsrc`
3. Updates `settings` object

### hub

Manage validators from Guardrails Hub.

**hub install:**
```bash
# Install latest version
guardrails hub install hub://guardrails/regex_match

# Install specific version
guardrails hub install hub://guardrails/regex_match@1.2.3

# Install multiple
guardrails hub install hub://guardrails/regex_match hub://guardrails/toxic_language
```

**hub list:**
```bash
# List installed validators
guardrails hub list
```

**hub search:**
```bash
# Search available validators
guardrails hub search regex
```

**hub uninstall:**
```bash
# Uninstall validator
guardrails hub uninstall hub://guardrails/regex_match
```

**Implementation:** `guardrails/cli/hub/`

### create

Create a Guard configuration file.

```bash
# Create with validators
guardrails create \
  --validators=hub://guardrails/regex_match \
  --guard-name=my-guard \
  --output=config.py

# Interactive mode
guardrails create --interactive
```

**Output (config.py):**
```python
from guardrails import Guard
from guardrails.hub import RegexMatch

my_guard = Guard(name="my-guard").use(
    RegexMatch(regex=r"...")
)
```

**Implementation:** `guardrails/cli/create.py`

### start

Start the Guardrails server.

```bash
# Basic start
guardrails start

# With configuration
guardrails start --config=config.py --port=8000

# With environment file
guardrails start --env=.env --config=config.py

# With watch mode (for logs)
guardrails start --watch --config=config.py
```

**Options:**
- `--config`: Path to Guard configuration file
- `--port`: Server port (default: 8000)
- `--env`: Environment file with API keys
- `--watch`: Enable watch mode for logs

**Implementation:** `guardrails/cli/start.py`

**Requirements:**
- Installs `guardrails-api` package if not present
- Loads Guards from config file
- Starts Flask/Gunicorn server

### validate

Validate data using a Guard from command line.

```bash
# Validate string
guardrails validate \
  --guards=config.py:my_guard \
  --data="test string"

# Validate from file
guardrails validate \
  --guards=config.py:my_guard \
  --file=input.json

# With LLM
guardrails validate \
  --guards=config.py:my_guard \
  --prompt="Generate..." \
  --llm=openai:gpt-4
```

**Implementation:** `guardrails/cli/validate.py`

### watch

Run Guard in watch mode (re-run on file changes).

```bash
guardrails watch \
  --guards=config.py:my_guard \
  --files="*.py" \
  --command="python script.py"
```

**Implementation:** `guardrails/cli/watch.py`

### version

Display version information.

```bash
guardrails version
```

**Output:**
```
Guardrails AI: 0.6.7
Python: 3.12.3
```

**Implementation:** `guardrails/cli/version.py`

## 🌐 Server Architecture

### Overview

The Guardrails server provides a REST API for Guards, enabling:
- Remote Guard execution
- OpenAI-compatible API
- Multi-Guard deployment
- Centralized validation service

**Package:** `guardrails-api` (separate package)

**Integration:** `guardrails/cli/server/`

### Server Modes

**1. Development Mode:**
```bash
guardrails start --config=config.py
```
- Uses Flask development server
- Hot reload on code changes
- Single process

**2. Production Mode (Docker):**
```dockerfile
FROM python:3.11
RUN pip install guardrails-ai[api]
COPY config.py /app/
CMD ["guardrails", "start", "--config=/app/config.py", "--port=8000"]
```
- Uses Gunicorn WSGI server
- Multiple workers
- Production-ready

### Configuration File

**config.py:**
```python
from guardrails import Guard
from guardrails.hub import RegexMatch, ToxicLanguage

# Define Guards
phone_guard = Guard(name="phone-validator").use(
    RegexMatch(regex=r"\d{3}-\d{3}-\d{4}")
)

content_guard = Guard(name="content-filter").use_many(
    ToxicLanguage(threshold=0.5),
    # ... other validators
)

# Export Guards (server discovers them)
__all__ = ["phone_guard", "content_guard"]
```

### Server Endpoints

**Validate Endpoint:**
```
POST /guards/{guard_name}/validate
Content-Type: application/json

{
  "value": "data to validate",
  "metadata": {"key": "value"}
}
```

**Response:**
```json
{
  "validated_output": "validated data",
  "validation_passed": true,
  "error": null
}
```

**LLM Endpoint:**
```
POST /guards/{guard_name}/execute
Content-Type: application/json

{
  "llm_api": "openai",
  "prompt": "Generate...",
  "model": "gpt-4",
  "llm_api_kwargs": {...}
}
```

**OpenAI-Compatible Endpoint:**
```
POST /guards/{guard_name}/openai/v1/chat/completions
Content-Type: application/json

{
  "model": "gpt-4",
  "messages": [...]
}
```

**Health Check:**
```
GET /health
```

**List Guards:**
```
GET /guards
```

### Client Integration

**Python Client:**
```python
import guardrails as gr

# Configure to use server
gr.settings.use_server = True
gr.settings.api_base_url = "http://localhost:8000"

# Use Guards from server
guard = gr.Guard(name="phone-validator")
outcome = guard.validate("123-456-7890")
```

**OpenAI SDK:**
```python
import openai

openai.base_url = "http://localhost:8000/guards/content-filter/openai/v1/"

completion = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Generate..."}]
)
```

**cURL:**
```bash
curl -X POST http://localhost:8000/guards/phone-validator/validate \
  -H "Content-Type: application/json" \
  -d '{"value": "123-456-7890"}'
```

## 🔧 CLI Implementation Details

### CLI Structure

```
guardrails/cli/
├── __init__.py              # Main CLI app
├── guardrails.py            # CLI group definition
├── configure.py             # Configure command
├── create.py                # Create command
├── start.py                 # Start command
├── validate.py              # Validate command
├── watch.py                 # Watch command
├── version.py               # Version command
├── logger.py                # CLI logging
├── telemetry.py             # Usage telemetry
├── hub/                     # Hub commands
│   ├── __init__.py
│   ├── install.py
│   ├── list.py
│   ├── search.py
│   └── utils.py
└── server/                  # Server utilities
    ├── __init__.py
    ├── hub_client.py        # Hub API client
    └── module_manifest.py   # Module management
```

### Typer Framework

Commands are defined using Typer:

```python
import typer
from guardrails.cli.guardrails import guardrails

@guardrails.command()
def my_command(
    arg: str = typer.Argument(..., help="Required argument"),
    option: int = typer.Option(default=0, help="Optional flag")
):
    """Command description."""
    # Implementation
    pass
```

### Rich Console Output

CLI uses `rich` for beautiful output:

```python
from rich.console import Console
from rich.table import Table

console = Console()

# Pretty printing
console.print("[bold green]Success![/bold green]")

# Tables
table = Table(title="Installed Validators")
table.add_column("Name")
table.add_column("Version")
table.add_row("regex_match", "1.2.3")
console.print(table)

# Progress bars
from rich.progress import track

for item in track(items, description="Processing..."):
    process(item)
```

### Error Handling

```python
from guardrails.cli.logger import logger

try:
    # Command logic
    pass
except Exception as e:
    logger.error(f"Command failed: {e}")
    raise typer.Exit(code=1)
```

## 📊 Server Implementation Details

### Server Startup Flow

```
1. Load configuration file (config.py)
2. Discover Guard instances
3. Register Guards with API
4. Start Flask/Gunicorn server
5. Register routes for each Guard
6. Start health check endpoint
```

**Code path:**
```
guardrails/cli/start.py:start()
  → import guardrails_api
  → guardrails_api.cli.start:start_api()
    → load config
    → create Flask app
    → register guards
    → start server
```

### Guard Discovery

Server discovers Guards from config file:

```python
# Server scans module
import importlib
import inspect

module = importlib.import_module("config")

# Find Guard instances
guards = {}
for name, obj in inspect.getmembers(module):
    if isinstance(obj, Guard):
        guards[obj.name or name] = obj
```

### Request Handling

**Validation Request:**
```
1. Receive POST /guards/{name}/validate
2. Extract guard by name
3. Parse request body
4. Call guard.validate()
5. Serialize ValidationOutcome
6. Return JSON response
```

**LLM Request:**
```
1. Receive POST /guards/{name}/execute
2. Extract guard and LLM config
3. Call guard(llm_api=..., ...)
4. Handle streaming if requested
5. Return result
```

### Authentication

Server can use JWT authentication:

```python
# In config.py
from guardrails.hub_token import get_jwt_token

# Guards are protected
token = get_jwt_token()
```

**Request with auth:**
```bash
curl -X POST http://localhost:8000/guards/my-guard/validate \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"value": "..."}'
```

## 🐳 Docker Deployment

### Dockerfile Example

```dockerfile
FROM python:3.11-slim

# Install Guardrails
RUN pip install guardrails-ai[api]

# Copy configuration
WORKDIR /app
COPY config.py /app/
COPY .env /app/

# Install validators
RUN guardrails hub install hub://guardrails/regex_match

# Expose port
EXPOSE 8000

# Start server
CMD ["guardrails", "start", "--config=/app/config.py", "--port=8000"]
```

### Docker Compose

```yaml
version: '3.8'
services:
  guardrails:
    build: .
    ports:
      - "8000:8000"
    environment:
      - GUARDRAILS_HUB_API_KEY=${HUB_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./config.py:/app/config.py
    command: guardrails start --config=/app/config.py --port=8000
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guardrails
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: guardrails
        image: guardrails:latest
        ports:
        - containerPort: 8000
        env:
        - name: GUARDRAILS_HUB_API_KEY
          valueFrom:
            secretKeyRef:
              name: guardrails-secrets
              key: hub-api-key
```

## 🔍 Debugging and Logging

### CLI Logging

```bash
# Enable debug logs
export GUARDRAILS_LOG_LEVEL=DEBUG
guardrails start --config=config.py

# Or via CLI
guardrails --log-level=DEBUG start
```

### Server Logging

```python
# In config.py
import logging
from guardrails import configure_logging

configure_logging(level=logging.DEBUG)
```

### Watch Mode

```bash
# Enable watch mode for detailed logs
guardrails start --watch --config=config.py
```

**Features:**
- Real-time log streaming
- Request/response logging
- Validator execution logs
- Performance metrics

## 🚀 Production Best Practices

### 1. Use Gunicorn

```bash
# Production server
gunicorn \
  --workers 4 \
  --bind 0.0.0.0:8000 \
  --timeout 120 \
  guardrails_api.app:create_app
```

### 2. Environment Variables

```bash
# .env file
GUARDRAILS_HUB_API_KEY=xxx
OPENAI_API_KEY=xxx
GUARDRAILS_LOG_LEVEL=INFO
GUARDRAILS_ENABLE_METRICS=true
```

### 3. Load Balancing

```nginx
upstream guardrails {
    server guardrails:8000;
    server guardrails:8001;
    server guardrails:8002;
}

server {
    location / {
        proxy_pass http://guardrails;
    }
}
```

### 4. Health Checks

```bash
# Health check endpoint
curl http://localhost:8000/health

# Response
{"status": "healthy", "guards": ["guard1", "guard2"]}
```

### 5. Monitoring

- Use OpenTelemetry for tracing
- Monitor response times
- Track validation success rates
- Alert on errors

## 📚 CLI Reference

| Command | Description | Example |
|---------|-------------|---------|
| `configure` | Configure Guardrails | `guardrails configure` |
| `hub install` | Install validator | `guardrails hub install hub://...` |
| `hub list` | List validators | `guardrails hub list` |
| `create` | Create Guard config | `guardrails create --guard-name=x` |
| `start` | Start server | `guardrails start --config=x.py` |
| `validate` | Validate data | `guardrails validate --data=x` |
| `watch` | Watch mode | `guardrails watch --files="*.py"` |
| `version` | Show version | `guardrails version` |

## 📚 Next Steps

- [Hub Integration](./06-hub-integration.md) - Validator management
- [Development Guide](./08-development-guide.md) - Contributing
- [API Structure](./04-api-structure.md) - Programmatic usage

---

The CLI and server provide flexible deployment options for Guardrails in any environment.
