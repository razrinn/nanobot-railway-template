# Agent Guidelines for nanobot-railway-template

This is a Python web application template for deploying [nanobot](https://github.com/nano-bot/nanobot) on Railway with a web-based config UI and status dashboard.

## Project Overview

- **Framework**: Starlette (ASGI) + uvicorn
- **Template Engine**: Jinja2
- **Config Validation**: Pydantic
- **Main File**: `server.py`
- **Dependencies**: `requirements.txt` (uses `nanobot-ai` from PyPI)

## Build & Run Commands

### Local Development

```bash
# Install dependencies
pip install -r requirements.txt
pip install nanobot-ai

# Run the server
python server.py

# Or with uvicorn (hot reload)
uvicorn server:app --reload --port 8080
```

### Docker

```bash
# Build
docker build -t nanobot .

# Run
docker run --rm -it -p 8080:8080 -e PORT=8080 -e ADMIN_PASSWORD=changeme -v nanobot-data:/data nanobot
```

### Railway Deployment

Click the deploy button in README.md or connect this repo to Railway.

## Testing

No formal test suite exists for this project. To add tests:

```bash
# Install test dependencies
pip install pytest pytest-asyncio httpx

# Run all tests
pytest

# Run a single test file
pytest tests/test_server.py

# Run a single test
pytest tests/test_server.py::test_health_endpoint
```

## Code Style Guidelines

### General

- Use **Python 3.12** features (see Dockerfile)
- Follow [PEP 8](https://peps.python.org/pep-0008/) with 100 character line limit
- Use **Black** for formatting: `black .`
- Use **Ruff** for linting: `ruff check .`
- Type hints are encouraged but not strictly enforced

### Imports

```python
# Standard library first, then third-party, then local
import asyncio
import json
import os
from pathlib import Path

from starlette.applications import Starlette
from starlette.responses import JSONResponse

from nanobot.config.loader import load_config
from nanobot.config.schema import Config
```

### Naming Conventions

- **Classes**: `PascalCase` (e.g., `GatewayManager`)
- **Functions/variables**: `snake_case` (e.g., `get_status`, `api_key`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `ADMIN_PASSWORD`)
- **Async functions**: Prefix with `async_` only if needed for clarity

### Type Hints

Use type hints for function signatures and instance variables:

```python
class GatewayManager:
    def __init__(self):
        self.process: asyncio.subprocess.Process | None = None
        self.state: str = "stopped"
        self.logs: deque[str] = deque(maxlen=500)

    async def start(self) -> None:
        ...
```

### Error Handling

- Use specific exception types when possible
- Return proper HTTP status codes in JSON responses
- Log errors before returning error responses
- Never expose sensitive data (secrets, API keys) in error messages

```python
async def api_config_put(request: Request):
    try:
        body = await request.json()
    except Exception:
        return JSONResponse({"error": "Invalid JSON"}, status_code=400)

    try:
        new_config = Config.model_validate(snake_data)
    except Exception as e:
        return JSONResponse({"error": f"Validation error: {e}"}, status_code=400)
```

### Async/Await

- Always use `async def` for functions that await
- Use `asyncio.create_task()` for fire-and-forget background tasks
- Properly handle `asyncio.CancelledError` in long-running tasks

### Pydantic Models

Use Pydantic v2 for config validation:

```python
from pydantic import BaseModel, Field

class Config(BaseModel):
    providers: dict[str, ProviderConfig] = Field(default_factory=dict)
    channels: dict[str, ChannelConfig] = Field(default_factory=dict)
```

### Template Files

- Location: `templates/`
- Use Jinja2 syntax
- Keep logic minimal; do processing in `server.py`

### Security

- Never commit secrets or API keys
- Use environment variables for sensitive configuration
- Mask secrets in API responses (see `mask_secrets()` function)
- Implement proper authentication (see `BasicAuthBackend`)

## File Structure

```
.
├── server.py          # Main application (routes, auth, gateway management)
├── requirements.txt   # Python dependencies
├── Dockerfile         # Railway-compatible container definition
├── start.sh          # Container startup script
├── templates/
│   └── index.html    # Web UI template
└── README.md         # Project documentation
```

## Key Functions

- `server.py:47` - `BasicAuthBackend`: HTTP Basic authentication
- `server.py:78` - `GatewayManager`: Manages nanobot gateway subprocess
- `server.py:158` - `mask_secrets()`: Masks sensitive fields in config responses
- `server.py:186` - `merge_secrets()`: Preserves existing secrets when updating config
- `server.py:218` - `api_config_put()`: Validates and saves configuration

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8080` | Web server port |
| `ADMIN_USERNAME` | `admin` | Basic auth username |
| `ADMIN_PASSWORD` | *(generated)* | Basic auth password |
| `NANOBOT_*` | - | nanobot config (prefixed) |

## Adding New Features

1. Add new routes to the `routes` list in `server.py`
2. Use existing auth pattern: call `require_auth(request)` and return early if unauthorized
3. Return `JSONResponse` for API endpoints
4. Add corresponding frontend changes in `templates/index.html`
