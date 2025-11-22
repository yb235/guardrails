# Hub Integration

This document explains how Guardrails integrates with the Guardrails Hub for validator management and distribution.

## 🌐 What is Guardrails Hub?

Guardrails Hub is a centralized repository of pre-built validators that can be:
- Discovered and searched
- Installed locally
- Used in Guards
- Shared with the community

**Hub Website:** https://hub.guardrailsai.com/

**Location in Code:** `guardrails/hub/`

## 🏗️ Hub Architecture

### Components

```
┌─────────────────────────────────────────────────────────┐
│                    Guardrails Hub                       │
│                  (hub.guardrailsai.com)                │
│  - Validator Registry                                   │
│  - Package Repository                                   │
│  - Authentication                                        │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│              Hub Client (local)                         │
│  guardrails/hub/                                        │
│  - Discovery                                            │
│  - Installation                                         │
│  - Version Management                                    │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│           Validator Packages                            │
│  ~/.guardrails/validators/                             │
│  - Installed validators                                 │
│  - Dependencies                                         │
│  - Metadata                                             │
└─────────────────────────────────────────────────────────┘
```

## 📦 Validator Package Structure

### Package Format

Hub validators are Python packages with specific structure:

```
validator-package/
├── validator/
│   ├── __init__.py
│   ├── validator.py      # Main validator class
│   └── utils.py          # Helper functions
├── tests/
│   └── test_validator.py
├── setup.py              # Package metadata
├── requirements.txt      # Dependencies
└── README.md            # Documentation
```

### Validator Metadata

Each validator includes metadata:

```python
# In validator.py
class MyValidator(Validator):
    """
    Validator description.
    
    **Key Properties**
    | Property | Description |
    | --- | --- |
    | Name | my_validator |
    | Type | string |
    | Supported Data Types | string |
    """
    
    rail_alias = "my-validator"
```

## 🔧 Installation Process

### Installation Flow

```
1. User runs: guardrails hub install hub://org/validator
                ↓
2. Parse validator URI
   - Org: org
   - Name: validator
   - Version: latest (or specified)
                ↓
3. Query Hub API for validator metadata
   - Get download URL
   - Get dependencies
   - Get version info
                ↓
4. Download validator package
   - From Hub registry
   - Verify integrity
                ↓
5. Install to ~/.guardrails/validators/
   - Extract package
   - Install dependencies
   - Register validator
                ↓
6. Make available for import
   - Add to Python path
   - Cache metadata
```

### Code Path

**Location:** `guardrails/hub/install.py`

```python
def install(
    uri: str,               # hub://org/validator
    upgrade: bool = False,  # Upgrade if already installed
    quiet: bool = False,    # Suppress output
    install_local_models: bool = True  # Install ML models
):
    """Install a validator from Hub."""
    
    # 1. Parse URI
    namespace, name, version = parse_hub_uri(uri)
    
    # 2. Check if already installed
    if is_installed(name) and not upgrade:
        print(f"{name} is already installed")
        return
    
    # 3. Fetch from Hub
    package_info = fetch_package_info(namespace, name, version)
    
    # 4. Download package
    download_path = download_package(
        package_info.download_url,
        name,
        version
    )
    
    # 5. Install dependencies
    install_dependencies(package_info.requirements)
    
    # 6. Register validator
    register_validator(name, version, download_path)
    
    # 7. Install models (if ML-based)
    if install_local_models and package_info.has_models:
        install_models(name)
    
    print(f"✓ Installed {name}@{version}")
```

### Installation Location

Validators are installed to:

```
~/.guardrails/
├── validators/
│   ├── hub/
│   │   ├── guardrails/
│   │   │   ├── regex_match/
│   │   │   │   ├── 1.2.3/
│   │   │   │   │   ├── validator/
│   │   │   │   │   └── metadata.json
│   │   │   └── toxic_language/
│   │   └── other_org/
│   └── manifest.json  # Registry of installed validators
└── config.json        # Hub configuration
```

## 🔍 Validator Discovery

### Package Service

**Location:** `guardrails/hub/validator_package_service.py`

```python
class ValidatorPackageService:
    """Manages installed validator packages."""
    
    def get_installed_validators(self) -> List[str]:
        """List all installed validators."""
        manifest = self._load_manifest()
        return list(manifest.keys())
    
    def get_validator_path(self, name: str) -> Path:
        """Get installation path for validator."""
        manifest = self._load_manifest()
        if name not in manifest:
            raise ValueError(f"Validator {name} not installed")
        return Path(manifest[name]["path"])
    
    def get_validator_version(self, name: str) -> str:
        """Get installed version."""
        manifest = self._load_manifest()
        return manifest[name]["version"]
```

### Import Resolution

When importing from hub:

```python
from guardrails.hub import RegexMatch
```

**Resolution flow:**
```
1. Import request for guardrails.hub.RegexMatch
                ↓
2. Hub __init__.py checks manifest
                ↓
3. Finds validator in ~/.guardrails/validators/
                ↓
4. Adds to Python path
                ↓
5. Imports validator class
                ↓
6. Returns class to user
```

**Code:** `guardrails/hub/__init__.py`

```python
import importlib
import sys
from pathlib import Path

def __getattr__(name: str):
    """
    Lazy loading of hub validators.
    Called when accessing guardrails.hub.<ValidatorName>
    """
    # Check if validator is installed
    service = ValidatorPackageService()
    
    if not service.is_installed(name):
        raise ImportError(
            f"Validator {name} not installed. "
            f"Install with: guardrails hub install hub://guardrails/{name}"
        )
    
    # Get validator path
    validator_path = service.get_validator_path(name)
    
    # Add to path if needed
    if str(validator_path) not in sys.path:
        sys.path.insert(0, str(validator_path))
    
    # Import validator module
    module = importlib.import_module(f"{name}.validator")
    
    # Return validator class
    return getattr(module, name)
```

## 🔐 Authentication

### JWT Tokens

Hub uses JWT for authentication:

**Location:** `guardrails/hub_token/`

```python
from guardrails.hub_token import get_jwt_token

# Get token (from ~/.guardrailsrc or env)
token = get_jwt_token()

# Use in API requests
headers = {"Authorization": f"Bearer {token}"}
```

### Token Storage

Tokens are stored in:

```
~/.guardrailsrc
```

**Format:**
```json
{
  "token": "eyJ...",
  "enable_metrics": true,
  "enable_remote_inferencing": true
}
```

### Token Configuration

```bash
# Via CLI
guardrails configure

# Via environment
export GUARDRAILS_TOKEN="your-token"

# Via Python
from guardrails import settings
settings.hub_api_key = "your-token"
```

## 📡 Remote Inference

### What is Remote Inference?

Some validators use ML models that:
- Are too large to run locally
- Require specialized hardware (GPU)
- Benefit from centralized deployment

These validators call Hub's inference API.

**Location:** `guardrails/remote_inference/`

### Remote Inference Flow

```
1. Validator.validate() called
                ↓
2. Check if validator uses remote inference
                ↓
3. Send request to Hub inference API
   - Value to validate
   - Validator parameters
   - Authentication token
                ↓
4. Hub runs model inference
   - Load model
   - Run prediction
   - Return result
                ↓
5. Parse response
   - PassResult or FailResult
                ↓
6. Return to Guard
```

### Code Example

```python
from guardrails.remote_inference import remote_inference

class MLValidator(Validator):
    """Validator using remote ML model."""
    
    def validate(self, value, metadata):
        # Call remote inference
        result = remote_inference(
            validator_name=self.rail_alias,
            value=value,
            metadata=metadata,
            model_params=self.get_model_params(),
            token=get_jwt_token()
        )
        
        return result
```

**Implementation:** `guardrails/remote_inference/remote_inference.py`

```python
def remote_inference(
    validator_name: str,
    value: Any,
    metadata: Dict,
    model_params: Dict,
    token: str
) -> ValidationResult:
    """
    Call Hub inference API for validation.
    """
    url = f"{HUB_INFERENCE_ENDPOINT}/validate"
    
    payload = {
        "validator": validator_name,
        "value": value,
        "metadata": metadata,
        "params": model_params
    }
    
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    
    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()
    
    data = response.json()
    
    if data["passed"]:
        return PassResult()
    else:
        return FailResult(
            error_message=data["error"],
            fix_value=data.get("fix_value")
        )
```

## 📊 Hub Telemetry

### Telemetry Data

Hub collects anonymized telemetry (opt-in):

**Location:** `guardrails/hub_telemetry/`

**Data collected:**
- Validator usage counts
- Validation success/failure rates
- Performance metrics
- Error types

**NOT collected:**
- Actual validation data
- User inputs/outputs
- Sensitive information

### Telemetry Flow

```python
from guardrails.hub_telemetry import trace

@trace(validator_name="my_validator")
def validate(self, value, metadata):
    """Validation with telemetry."""
    # Validation logic
    return result
```

**What happens:**
```
1. Decorator wraps validate()
2. Records start time
3. Execute validation
4. Record result (pass/fail)
5. Send telemetry to Hub (async)
6. Return result
```

### Opt-out

```bash
# Via configure
guardrails configure
# Select "no" for telemetry

# Via environment
export GUARDRAILS_ENABLE_METRICS=false
```

## 🔄 Validator Versioning

### Version Specification

```bash
# Latest version
guardrails hub install hub://guardrails/regex_match

# Specific version
guardrails hub install hub://guardrails/regex_match@1.2.3

# Version range (in requirements)
regex-match>=1.2.0,<2.0.0
```

### Version Management

**Location:** Handled by `validator_package_service.py`

```python
# Check installed version
service = ValidatorPackageService()
version = service.get_validator_version("regex_match")

# Upgrade to latest
install("hub://guardrails/regex_match", upgrade=True)
```

### Compatibility

Validators declare compatibility:

```python
# In setup.py or metadata
{
    "name": "regex-match",
    "version": "1.2.3",
    "guardrails_version": ">=0.5.0,<0.7.0",
    "python_version": ">=3.9"
}
```

## 🎨 Creating Hub Validators

### Validator Template

```python
from guardrails import Validator, register_validator
from guardrails.classes import PassResult, FailResult

@register_validator(name="my_validator", data_type="string")
class MyValidator(Validator):
    """
    Validator description.
    
    **Supported data types:** string
    **Requirements:** None
    """
    
    def __init__(
        self,
        param1: str,
        on_fail: str = "noop"
    ):
        super().__init__(on_fail=on_fail)
        self.param1 = param1
    
    def validate(self, value: str, metadata: dict) -> ValidationResult:
        """Main validation logic."""
        if self.check_value(value):
            return PassResult()
        
        return FailResult(
            error_message="Validation failed",
            fix_value=self.suggest_fix(value)
        )
    
    def check_value(self, value: str) -> bool:
        """Implement validation logic."""
        return True
```

### Publishing to Hub

1. **Create validator package**
2. **Write tests**
3. **Document validator**
4. **Submit to Hub**

```bash
# Package validator
python setup.py sdist

# Submit (requires Hub account)
guardrails hub publish ./dist/my-validator-1.0.0.tar.gz
```

## 🔍 Hub API Client

### Client Implementation

**Location:** `guardrails/cli/server/hub_client.py`

```python
class HubClient:
    """Client for Guardrails Hub API."""
    
    def __init__(self, token: str = None):
        self.token = token or get_jwt_token()
        self.base_url = "https://api.guardrailsai.com"
    
    def search_validators(self, query: str) -> List[dict]:
        """Search for validators."""
        response = requests.get(
            f"{self.base_url}/validators/search",
            params={"q": query},
            headers=self._auth_headers()
        )
        return response.json()
    
    def get_validator_info(
        self,
        namespace: str,
        name: str,
        version: str = "latest"
    ) -> dict:
        """Get validator metadata."""
        url = f"{self.base_url}/validators/{namespace}/{name}/{version}"
        response = requests.get(url, headers=self._auth_headers())
        return response.json()
    
    def download_validator(
        self,
        namespace: str,
        name: str,
        version: str
    ) -> bytes:
        """Download validator package."""
        url = f"{self.base_url}/validators/{namespace}/{name}/{version}/download"
        response = requests.get(url, headers=self._auth_headers())
        return response.content
    
    def _auth_headers(self) -> dict:
        return {"Authorization": f"Bearer {self.token}"}
```

## 🛠️ Local Development

### Testing Validators Locally

```python
# Test validator without installing
from my_validator import MyValidator

validator = MyValidator(param="value")

# Test validation
result = validator.validate("test value", {})

assert result.passed
```

### Local Package Installation

```bash
# Install from local path
pip install -e ./my-validator/

# Use in Guardrails
from my_validator import MyValidator
guard = Guard().use(MyValidator())
```

## 📚 Hub Commands Reference

| Command | Description | Example |
|---------|-------------|---------|
| `hub install` | Install validator | `guardrails hub install hub://org/name` |
| `hub list` | List installed | `guardrails hub list` |
| `hub search` | Search Hub | `guardrails hub search regex` |
| `hub uninstall` | Remove validator | `guardrails hub uninstall name` |
| `hub publish` | Publish validator | `guardrails hub publish package.tar.gz` |

## 🔒 Security Considerations

### Package Verification

- Packages are signed by Hub
- Checksums verified on download
- Only trusted namespaces allowed

### Sandboxing

- Validators run in main process by default
- Can be isolated with `run_in_separate_process = True`
- Remote inference isolates ML models

### API Keys

- Tokens stored securely in `~/.guardrailsrc`
- Environment variables supported
- Never committed to code

## 🎯 Best Practices

### 1. Pin Versions in Production

```python
# requirements.txt
guardrails-ai==0.6.7
regex-match==1.2.3  # Pin specific versions
```

### 2. Test Before Deployment

```python
# Test installed validators
from guardrails.hub import RegexMatch

validator = RegexMatch(regex=r"\d+")
assert validator.validate("123", {}).passed
```

### 3. Monitor Remote Inference

```python
# Check if using remote inference
if validator.uses_remote_inference:
    # Ensure token is valid
    # Monitor API quotas
    pass
```

### 4. Cache Validators

```python
# Reuse validator instances
validator = RegexMatch(regex=r"\d+")

# Use in multiple guards
guard1 = Guard().use(validator)
guard2 = Guard().use(validator)
```

## 📚 Next Steps

- [Module Reference](./07-module-reference.md) - Detailed module breakdown
- [Development Guide](./08-development-guide.md) - Contributing validators
- [API Structure](./04-api-structure.md) - Using validators in code

---

The Hub ecosystem makes it easy to discover, share, and reuse validators across the community.
