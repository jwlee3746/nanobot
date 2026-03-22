# 01. Project Structure

## Directory Layout

```
nanobot/
├── nanobot/                        # Main Python package
│   ├── __init__.py                 # Version, logo constants
│   ├── __main__.py                 # Entry point: python -m nanobot
│   │
│   ├── agent/                      # [Core] Agent processing engine
│   │   ├── loop.py                 # AgentLoop — main message processing loop
│   │   ├── context.py              # ContextBuilder — system prompt assembly
│   │   ├── memory.py               # MemoryStore + MemoryConsolidator
│   │   ├── skills.py               # SkillsLoader — skill discovery & loading
│   │   ├── subagent.py             # SubagentManager — background task spawning
│   │   └── tools/                  # Tool subsystem
│   │       ├── base.py             # Tool ABC with JSON Schema validation
│   │       ├── registry.py         # ToolRegistry — dynamic tool management
│   │       ├── filesystem.py       # ReadFile, WriteFile, EditFile, ListDir
│   │       ├── shell.py            # ExecTool — shell command execution
│   │       ├── web.py              # WebSearch, WebFetch
│   │       ├── mcp.py              # MCP server integration (stdio/SSE/HTTP)
│   │       ├── message.py          # MessageTool — direct channel messaging
│   │       ├── spawn.py            # SpawnTool — subagent creation
│   │       └── cron.py             # CronTool — scheduled task management
│   │
│   ├── bus/                        # [Core] Event-driven message queue
│   │   ├── events.py               # InboundMessage, OutboundMessage dataclasses
│   │   └── queue.py                # MessageBus — async inbound/outbound queues
│   │
│   ├── channels/                   # [Integration] Chat platform adapters
│   │   ├── base.py                 # BaseChannel ABC
│   │   ├── manager.py              # ChannelManager — lifecycle & routing
│   │   ├── registry.py             # Dynamic channel discovery (pkgutil + entry_points)
│   │   ├── telegram.py             # Telegram Bot API
│   │   ├── discord.py              # Discord Bot
│   │   ├── slack.py                # Slack (thread isolation, mrkdwn)
│   │   ├── whatsapp.py             # WhatsApp via Baileys bridge
│   │   ├── feishu.py               # Feishu (Lark)
│   │   ├── dingtalk.py             # DingTalk
│   │   ├── qq.py                   # QQ
│   │   ├── wecom.py                # WeCom (WeChat Work)
│   │   ├── matrix.py               # Matrix protocol (E2E)
│   │   ├── email.py                # Email (IMAP polling)
│   │   └── mochat.py               # MoChat
│   │
│   ├── providers/                  # [Integration] LLM provider abstractions
│   │   ├── base.py                 # LLMProvider ABC, LLMResponse, ToolCallRequest
│   │   ├── registry.py             # ProviderSpec metadata & auto-detection
│   │   ├── litellm_provider.py     # LiteLLM-based universal provider
│   │   ├── azure_openai_provider.py# Direct Azure OpenAI API
│   │   ├── custom_provider.py      # OpenAI-compatible endpoints
│   │   ├── openai_codex_provider.py# OAuth-based Codex provider
│   │   └── transcription.py        # Groq Whisper audio transcription
│   │
│   ├── config/                     # [Infrastructure] Configuration management
│   │   ├── schema.py               # Pydantic models (Config, AgentsConfig, etc.)
│   │   ├── loader.py               # load_config(), save_config(), migration
│   │   └── paths.py                # Path management (workspace, sessions, cron)
│   │
│   ├── session/                    # [Infrastructure] Conversation persistence
│   │   └── manager.py              # SessionManager, Session (JSONL storage)
│   │
│   ├── cron/                       # [Service] Task scheduling
│   │   ├── service.py              # CronService — cron/at/every scheduling
│   │   └── types.py                # CronJob, CronSchedule, CronStore
│   │
│   ├── heartbeat/                  # [Service] Periodic background checks
│   │   └── service.py              # HeartbeatService — HEARTBEAT.md monitoring
│   │
│   ├── security/                   # [Infrastructure] Security policies
│   │   └── network.py              # Private IP blocking, localhost validation
│   │
│   ├── skills/                     # [Content] Built-in skills
│   │   ├── clawhub/                # ClawHub skill marketplace
│   │   ├── cron/                   # Cron scheduling guide
│   │   ├── github/                 # GitHub integration
│   │   ├── memory/                 # Memory management
│   │   ├── skill-creator/          # Skill scaffolding
│   │   ├── summarize/              # Text summarization
│   │   ├── tmux/                   # Terminal multiplexer
│   │   └── weather/                # Weather information
│   │
│   ├── templates/                  # [Content] Workspace initialization
│   │   ├── AGENTS.md               # Agent persona template
│   │   ├── SOUL.md                 # Agent behavior guidelines
│   │   ├── USER.md                 # User profile template
│   │   ├── TOOLS.md                # Tool usage guidelines
│   │   └── memory/                 # Memory directory template
│   │
│   ├── cli/                        # [Interface] Command-line interface
│   │   └── commands.py             # Typer CLI (onboard, agent, gateway, status)
│   │
│   └── utils/                      # [Infrastructure] Shared utilities
│       ├── helpers.py              # Token estimation, time formatting, etc.
│       └── evaluator.py            # Expression evaluation for dynamic config
│
├── bridge/                         # WhatsApp Baileys bridge (TypeScript/Node.js)
├── tests/                          # Test suite (50+ test files, pytest)
├── docs/                           # Documentation
├── case/                           # Use case demonstrations
├── pyproject.toml                  # Project metadata, dependencies, build config
├── Dockerfile                      # Docker containerization
└── docker-compose.yml              # Multi-container orchestration
```

## Entry Points

### CLI Entry Point

```
pyproject.toml
  [project.scripts]
  nanobot = "nanobot.cli.commands:app"
              │
              ▼
nanobot/__main__.py
  from nanobot.cli.commands import app
              │
              ▼
nanobot/cli/commands.py
  app = typer.Typer(...)
  ├── @app.command() onboard   — Initialize config & workspace
  ├── @app.command() agent     — Interactive chat mode (CLI)
  ├── @app.command() gateway   — Server mode with channel integrations
  └── @app.command() status    — Display runtime info
```

### Two Operating Modes

| Mode | Command | Description |
|------|---------|-------------|
| **Agent** | `nanobot agent` | Single-session interactive CLI chat. Direct user ↔ agent conversation. |
| **Gateway** | `nanobot gateway` | Multi-channel server. Manages all enabled channels, cron, heartbeat concurrently. |

## Key Dependencies

| Package | Role |
|---------|------|
| `typer` | CLI framework |
| `litellm` | Universal LLM API proxy |
| `pydantic` / `pydantic-settings` | Configuration validation |
| `prompt_toolkit` | Interactive CLI input |
| `rich` | Terminal formatting |
| `loguru` | Structured logging |
| `httpx` | Async HTTP client |
| `croniter` | Cron expression parsing |
| `websockets` | WebSocket transport |
| `mcp` | Model Context Protocol SDK |
