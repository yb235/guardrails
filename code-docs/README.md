# Guardrails Code Documentation

Welcome to the Guardrails code documentation! This documentation is designed to help first-time users and contributors understand the codebase architecture, workflow, and key concepts.

## 📚 Documentation Structure

This documentation is organized into the following sections:

1. **[Architecture Overview](./01-architecture-overview.md)** - High-level system architecture and design patterns
2. **[Core Concepts](./02-core-concepts.md)** - Understanding Guards, Validators, and key abstractions
3. **[Workflow and Execution](./03-workflow-execution.md)** - How Guardrails processes requests from start to finish
4. **[API Structure](./04-api-structure.md)** - Public APIs, entry points, and how to use them
5. **[CLI and Server](./05-cli-server.md)** - Command-line interface and server deployment
6. **[Hub Integration](./06-hub-integration.md)** - Guardrails Hub, validators, and package management
7. **[Module Reference](./07-module-reference.md)** - Detailed breakdown of key modules and their responsibilities
8. **[Development Guide](./08-development-guide.md)** - How to contribute, test, and develop features
9. **[Testing Infrastructure](./09-testing-infrastructure.md)** - Understanding the test suite and how to write tests

## 🎯 Quick Start for Developers

### Understanding the Codebase

If you're new to Guardrails, we recommend reading the documentation in the following order:

1. Start with **Architecture Overview** to understand the high-level design
2. Read **Core Concepts** to understand Guards and Validators
3. Review **Workflow and Execution** to see how everything connects
4. Dive into **Module Reference** for specific implementation details

### For Contributors

If you want to contribute to Guardrails:

1. Read the **Development Guide** first
2. Understand the **Testing Infrastructure**
3. Review the **Module Reference** for the area you want to work on
4. Check the main [CONTRIBUTING.md](../CONTRIBUTING.md) for setup instructions

## 🏗️ What is Guardrails?

Guardrails is a Python framework that helps build reliable AI applications by:

1. **Running Input/Output Guards** - Detect, quantify, and mitigate specific types of risks in LLM applications
2. **Generating Structured Data** - Help LLMs produce structured, validated output that matches expected schemas

### Key Features

- **Validator Framework**: Extensible system for creating custom validation logic
- **Guard System**: Compose multiple validators into reusable Guards
- **Schema Support**: Pydantic, JSON Schema, and RAIL format support
- **LLM Integration**: Works with any LLM (OpenAI, Anthropic, local models, etc.)
- **Server Mode**: Deploy Guards as a REST API service
- **Hub Integration**: Access pre-built validators from Guardrails Hub
- **Async Support**: Full async/await support for high-performance applications
- **Streaming**: Support for streaming LLM responses with validation

## 🔧 Technology Stack

- **Language**: Python 3.9+
- **Key Dependencies**:
  - `pydantic` - Data validation and schema management
  - `openai` - LLM API integration
  - `typer` - CLI framework
  - `litellm` - Multi-provider LLM support
  - `langchain-core` - Integration with LangChain ecosystem
  - `opentelemetry` - Observability and tracing
  - `flask` - Server mode (via guardrails-api)

## 📁 Repository Structure

```
guardrails/
├── guardrails/           # Main source code
│   ├── actions/          # Failure handling actions (reask, refrain, filter)
│   ├── classes/          # Core data structures and models
│   ├── cli/              # Command-line interface
│   ├── hub/              # Hub integration and validator management
│   ├── run/              # Execution runners (sync, async, streaming)
│   ├── schema/           # Schema parsing and validation
│   ├── validators/       # Built-in validators
│   ├── guard.py          # Main Guard class
│   ├── async_guard.py    # Async Guard implementation
│   └── validator_base.py # Base validator class
├── tests/                # Test suite
│   ├── unit_tests/       # Unit tests
│   └── integration_tests/# Integration tests
├── docs/                 # User-facing documentation (Docusaurus)
├── code-docs/            # Internal code documentation (this folder)
└── pyproject.toml        # Project configuration
```

## 🚀 Getting Started with the Code

### Running Tests

```bash
# Install dependencies
make dev

# Run all tests
make test

# Run with coverage
make test-cov
```

### Code Quality

```bash
# Auto-format code
make autoformat

# Type checking
make type

# Linting
make lint
```

### Local Development

```bash
# Install in development mode
pip install -e .

# Or use make
make self-install
```

## 🤝 Community

- **GitHub Issues**: Report bugs or request features
- **Discord**: Join the [Guardrails Discord](https://discord.gg/gw4cR9QvYE)
- **Twitter**: Follow [@guardrails_ai](https://twitter.com/guardrails_ai)

## 📖 Additional Resources

- [Main Documentation](https://www.guardrailsai.com/docs) - User-facing documentation
- [Guardrails Hub](https://hub.guardrailsai.com/) - Pre-built validators
- [Blog](https://www.guardrailsai.com/blog) - Articles and updates
- [Contributing Guide](../CONTRIBUTING.md) - How to contribute

---

**Note**: This is internal code documentation for developers and contributors. For end-user documentation, visit [www.guardrailsai.com/docs](https://www.guardrailsai.com/docs).
