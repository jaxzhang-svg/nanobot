# AGENTS.md - nanobot Coding Guide

Build/lint/test commands and code style for agentic coders working on this repo.

## Build / Lint / Test Commands

```bash
pip install -e ".[dev]"
pytest
pytest tests/test_tool_validation.py -k test_validate_params_missing_required
pytest -v
ruff check nanobot/
ruff check --fix nanobot/
ruff format nanobot/
```

## Code Style Guidelines

**Python 3.11+** with modern syntax (`list[str]` not `List[str]`).

### Imports
Standard library first, then third-party, then local. Use `from __future__ import annotations` for forward references:
```python
import asyncio
from pathlib import Path
import typer
from loguru import logger
from nanobot.config.schema import Config

if TYPE_CHECKING:
    from nanobot.session.manager import SessionManager
```

### Naming Conventions
- Classes: `PascalCase` (e.g., `AgentLoop`, `BaseChannel`, `Tool`)
- Functions/methods: `snake_case` (e.g., `_register_default_tools`, `is_allowed`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `BOT_COMMANDS`)
- Private members: prefix with `_` (e.g., `_running`, `_handle_message`)

### Docstrings & Logging
```python
"""Module docstring."""
from loguru import logger

class AgentLoop:
    """The agent loop is the core processing engine."""

    async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
        """Process a single inbound message."""
        try:
            result = await self.bus.publish_outbound(msg)
        except Exception as e:
            logger.error(f"Error: {e}")
```

### Async/Await Patterns
Use `asyncio.wait_for` with timeouts for I/O:
```python
async def run(self) -> None:
    while self._running:
        try:
            msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)
            await self._process_message(msg)
        except asyncio.TimeoutError:
            continue
```

### Pydantic Models
Use `Field(default_factory=...)` for mutable defaults:
```python
class TelegramConfig(BaseModel):
    enabled: bool = False
    allow_from: list[str] = Field(default_factory=list)  # correct
    # NOT: allow_from: list[str] = []  # wrong
```

### Abstract Base Classes
```python
from abc import ABC, abstractmethod

class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None:
        """Start channel and begin listening."""
        pass
```

### Tools (inherit from Tool)
```python
from nanobot.agent.tools.base import Tool

class MyTool(Tool):
    @property
    def name(self) -> str:
        return "my_tool"

    @property
    def parameters(self) -> dict[str, Any]:
        return {"type": "object", "properties": {}, "required": []}

    async def execute(self, **kwargs: Any) -> str:
        return "ok"
```

### Providers (add to registry.py + schema.py)
```python
# registry.py:
ProviderSpec(name="myprovider", keywords=("myprovider",), env_key="MYPROVIDER_API_KEY")
# schema.py:
class ProvidersConfig(BaseModel):
    myprovider: ProviderConfig = Field(default_factory=ProviderConfig)
```

### Channels (inherit from BaseChannel)
```python
class MyChannel(BaseChannel):
    name = "mychannel"
    async def send(self, msg: OutboundMessage) -> None:
        pass
```
Forward via: `await self._handle_message(sender_id=str(user.id), chat_id=str(chat.id), content=text)`

### Config Access
```python
config = Config()
api_key = config.get_api_key(model="deepseek-chat")
workspace = config.workspace_path
```

## Testing
Use `pytest.mark.asyncio`:
```python
@pytest.mark.asyncio
async def test_async():
    tool = SampleTool()
    result = await tool.execute(query="test")
    assert result == "ok"
```

## Ruff Config
```toml
[tool.ruff]
line-length = 100
target-version = "py311"
```

## File Organization
```
nanobot/
├── agent/       # Core agent logic (loop, tools, skills)
├── channels/    # Chat platform integrations
├── providers/   # LLM provider registry
├── config/      # Pydantic config schemas
├── bus/         # Message routing
└── cli/         # Command-line interface
```
