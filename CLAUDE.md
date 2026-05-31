# CLAUDE.md

Guidance for Claude Code working in this repository.

## Project Overview

**multi-agent-system** is a hierarchical multi-agent system using Ollama for intelligent task routing. Integrates with Telegram. Includes Redis memory, email automation, code generation, vision, and web search.

## Architecture
```
Telegram Input → Router (gemma3:1b)
              ↓
         Specialists:
         ├─ Code (qwen2.5-coder)
         ├─ Vision (llama3.2-vision)
         ├─ Analysis (phi4)
         └─ Search (qwen2.5)
              ↓
         Synthesizer (aya:8b) → Telegram Output
```

## Build & Development

### Prerequisites
- Python 3.11+
- Poetry
- Ollama
- Redis (optional)
- Telegram Bot Token (@BotFather)
- 16GB+ RAM (24GB+ extended, 32GB+ maximum)
- NVIDIA GPU (optional, recommended)

### Setup
```bash
cd multi-agent-system && poetry install
poetry run agent-system init
cp .env.example .env  # Edit with TELEGRAM_BOT_TOKEN

ollama serve &
poetry run agent-system deploy-models essential  # ~16GB
```

### Running the Bot
```bash
poetry run agent-system run --polling      # Dev mode
poetry run agent-system run                # Webhook (production)
poetry run agent-system run-agent router   # Specific agent
```

## Testing

```bash
# Run all tests
poetry run pytest

# Run tests with coverage
poetry run pytest --cov=src/agent_system

# Run specific test file
poetry run pytest tests/unit/test_router.py

# Run integration tests only
poetry run pytest tests/integration/ -v

# Run with verbose output and logging
poetry run pytest -v -s
```

## Code Organization

```
src/agent_system/
├── main.py              # CLI entry point
├── config.py            # Configuration management
│
├── core/                # Core orchestration
│   ├── orchestrator.py  # Coordinates all agents
│   ├── router_agent.py  # Router (gemma3:1b)
│   └── specialist_base.py # Base class for agents
│
├── agents/              # Specialist agents
│   ├── code_specialist.py  # Code generation
│   ├── email_agent.py      # Email automation
│   ├── vision_agent.py     # Vision processing
│   ├── analysis_agent.py   # Deep analysis
│   ├── search_agent.py     # Web search
│   └── synthesis_agent.py  # Response formatting
│
├── telegram/            # Telegram integration
│   ├── bot.py           # Bot setup
│   ├── handlers.py      # Command handlers
│   ├── keyboards.py     # Keyboard layouts
│   └── callbacks.py     # Callback handlers
│
├── memory/              # Memory management
│   ├── manager.py       # Memory orchestration
│   ├── redis_client.py  # Redis connection
│   ├── vector_store.py  # Vector embeddings
│   └── session.py       # User sessions
│
├── automation/          # Automation modules
│   ├── email_client.py  # Email (IMAP/SMTP)
│   ├── code_executor.py # Code execution sandbox
│   ├── scheduler.py     # Task scheduling
│   └── notification.py  # Notifications
│
├── utils/               # Utilities
│   ├── logger.py        # Logging configuration
│   ├── validators.py    # Input validation
│   ├── helpers.py       # Helper functions
│   └── metrics.py       # Performance metrics
│
└── models/              # Model registry
    ├── schemas.py       # Pydantic schemas
    └── model_registry.py # Model capabilities
```

## Key Dependencies

| Package | Purpose |
|---------|---------|
| **python-telegram-bot** | Telegram integration |
| **httpx** | Async HTTP client |
| **python-dotenv** | Environment variables |
| **pydantic-settings** | Settings management |
| **typer** | CLI framework |
| **rich** | Terminal output formatting |
| **psutil** | System monitoring |

## Model Tiers

### Essential Tier (~16GB)
Minimum viable system for core functionality:
```bash
ollama pull gemma3:1b       # Router
ollama pull gemma3:4b       # Vision
ollama pull qwen2.5-coder:3b # Code (fast)
ollama pull qwen2.5-coder:7b # Code (complex)
ollama pull phi4:14b        # Analysis
```

### Extended Tier (~24GB)
Add multilingual and complex vision:
```bash
# Essential + these:
ollama pull llama3.2-vision:11b  # Complex vision
ollama pull minicpm-v:8b         # OCR/documents
ollama pull aya:8b               # Multilingual
ollama pull nous-hermes2:10.7b   # Advanced reasoning
```

### Maximum Tier (~32GB)
Full capability system:
```bash
# Extended + these:
ollama pull gemma3:12b       # Upgrade analysis
ollama pull qwen2.5:32b      # Advanced code
ollama pull deepseek-coder-v2:16b # Extreme code
ollama pull command-r:35b    # Ultra analysis
```

## Coding Conventions

**Code Style:** Python 3.11+ with async/await. Use Black, Ruff, isort. Comprehensive type hints required.

**File Organization:** Single responsibility per module. Max ~400 LOC per file. Descriptive names. Docstrings for classes and complex functions.

**Error Handling:** Typed exceptions (no bare `except`). Log with context. Telegram-friendly messages. Handle timeouts gracefully.

**Agent Development:**
1. Extend `SpecialistBase` from `core/specialist_base.py`
2. Implement `async def process(query: str) -> str`
3. Register in `orchestrator.py` and model registry
4. Create tests in `tests/unit/test_<agent_name>.py`

## Configuration

### Environment Variables (.env)
```bash
# Required
TELEGRAM_BOT_TOKEN=<your_token>

# Optional - Email
EMAIL_ADDRESS=your.email@gmail.com
EMAIL_PASSWORD=your_app_password

# Optional - Performance
MAX_CONCURRENT_REQUESTS=3
POLLING_INTERVAL=1.0
MODEL_TIER=essential  # or extended, maximum

# Optional - Redis
REDIS_URL=redis://localhost:6379
ENABLE_CACHING=true
```

## Common Development Tasks

### Add a new specialist agent
```python
# src/agent_system/agents/new_specialist.py
from .specialist_base import SpecialistBase

class NewSpecialist(SpecialistBase):
    MODEL_NAME = "appropriate-model:size"
    
    async def process(self, query: str) -> str:
        # Implementation
        return response
```

Then register in `core/orchestrator.py` and update `models/model_registry.py`.

### Add a Telegram command
1. Add handler in `telegram/handlers.py`
2. Register in `telegram/bot.py`
3. Update `/help` text
4. Add tests in `tests/`

### Debug agent routing
```bash
# Test router classification
poetry run python -c "
from src.agent_system.core.router_agent import RouterAgent
import asyncio
router = RouterAgent()
result = asyncio.run(router.classify_query('test query'))
print(result)
"
```

## Deployment

### Docker Deployment
```bash
# Build and run with Docker Compose
docker-compose up -d

# Check logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Systemd Service
```bash
# Create service file
sudo nano /etc/systemd/system/multi-agent-system.service

# Content:
[Unit]
Description=Multi-Agent System
After=network.target ollama.service

[Service]
Type=simple
User=youruser
WorkingDirectory=/path/to/multi-agent-system
ExecStart=/usr/bin/poetry run agent-system run --polling
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

# Enable and start
sudo systemctl enable multi-agent-system
sudo systemctl start multi-agent-system
```

## Monitoring

```bash
# View system metrics
curl http://localhost:8000/metrics

# Check model status
poetry run agent-system status

# View logs
journalctl -u multi-agent-system -f
```

## Troubleshooting

**"Connection refused" to Ollama:** Run `ollama serve &` and verify: `curl http://localhost:11434/api/tags`.

**Out of Memory:** Check loaded models: `ollama ps`. Use `MODEL_TIER=essential`. Reduce `MAX_CONCURRENT_REQUESTS=1`.

**Telegram connection issues:** Verify token: `poetry run python -c "from telegram import Bot; Bot(token='<token>').get_me()"`. Check network: `curl -I https://api.telegram.org`.

**Agent is slow:** Verify Router is gemma3:1b. Check MODEL_TIER and memory. Verify `ENABLE_CACHING=true` if using Redis.

## Performance Tips

```bash
# Optimal for 16GB RAM (Essential)
MODEL_TIER=essential \
MAX_CONCURRENT_REQUESTS=2 \
poetry run agent-system run --polling

# Optimal for 32GB+ RAM (Maximum)
MODEL_TIER=maximum \
MAX_CONCURRENT_REQUESTS=4 \
poetry run agent-system run --polling

# Enable Redis caching for faster responses
ENABLE_CACHING=true REDIS_URL=redis://localhost:6379 \
poetry run agent-system run --polling
```

## Testing & Quality

```bash
# Run full test suite
poetry run pytest tests/ -v

# Type checking
poetry run mypy src/

# Code formatting
poetry run black src/
poetry run isort src/

# Linting
poetry run ruff check src/

# All checks
poetry run pytest && poetry run mypy src/ && poetry run black --check src/ && poetry run ruff check src/
```

## Git Conventions

Commits: Action-oriented (`Add vision agent`, `Fix router timeout`). Branches: `feature/agent-name` or `fix/issue-description`. PRs: Include testing notes and model tier requirements.

## Available Skills & Agents

**BMAD-METHOD**: Agile AI development framework with expert agents for project planning, architecture, and workflow guidance. Use `bmad-help` for next steps. Install: `npx bmad-method install`.

## References

- **Python Telegram Bot**: https://python-telegram-bot.readthedocs.io/
- **Ollama API**: https://github.com/ollama/ollama/blob/main/docs/api.md
- **Poetry**: https://python-poetry.org/docs/
- **Asyncio**: https://docs.python.org/3/library/asyncio.html
- **Pydantic**: https://docs.pydantic.dev/
- **BMAD-METHOD**: https://github.com/bmad-code-org/BMAD-METHOD
