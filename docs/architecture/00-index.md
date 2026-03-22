# nanobot Architecture Documentation

> nanobot v0.1.4 — Ultra-lightweight Personal AI Assistant Framework

## Table of Contents

| # | Document | Description |
|---|---------|-------------|
| 01 | [Project Structure](./01-project-structure.md) | Directory layout, module organization, entry points |
| 02 | [Block Diagram](./02-block-diagram.md) | High-level system architecture, component relationships |
| 03 | [Sequence Diagrams](./03-sequence-diagrams.md) | Message processing, tool execution, memory consolidation flows |
| 04 | [Data Flow](./04-data-flow.md) | Data lifecycle, message routing, session & memory persistence |
| 05 | [Module Reference](./05-modules.md) | Detailed description of each module and its public interfaces |

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI / Gateway                           │
├─────────────┬──────────────────────────────────────┬────────────┤
│  Channels   │          Agent Core                  │ Providers  │
│  (Telegram, │  ┌────────────────────────────────┐  │ (Anthropic,│
│   Discord,  │  │  AgentLoop                     │  │  OpenAI,   │
│   Slack,    │  │  ┌──────────┐ ┌──────────────┐ │  │  DeepSeek, │
│   WhatsApp, │  │  │ Context  │ │ Tool         │ │  │  Gemini,   │
│   Feishu,   │◄─┤  │ Builder  │ │ Registry     │ ├─►│  Ollama,   │
│   DingTalk, │  │  └──────────┘ └──────────────┘ │  │  ...21+)   │
│   QQ,       │  │  ┌──────────┐ ┌──────────────┐ │  │            │
│   WeCom,    │  │  │ Memory   │ │ Subagent     │ │  │            │
│   Matrix,   │  │  │ System   │ │ Manager      │ │  │            │
│   Email,    │  │  └──────────┘ └──────────────┘ │  │            │
│   MoChat)   │  └────────────────────────────────┘  │            │
├─────────────┴──────────────┬───────────────────────┴────────────┤
│                      Message Bus                                │
│              (Async inbound/outbound queues)                    │
├──────────────────────────┬──────────────────────────────────────┤
│    Session Manager       │    Supporting Services               │
│    (JSONL persistence)   │    (Cron, Heartbeat, Skills, MCP)    │
└──────────────────────────┴──────────────────────────────────────┘
```

## Design Principles

1. **Lightweight** — Minimal dependencies, clean abstractions, no unnecessary complexity
2. **Decoupled** — Message bus separates channels from agent core
3. **Extensible** — Plugin channels, pluggable providers, MCP tool integration
4. **Resilient** — Retry logic, graceful degradation, memory consolidation fallbacks
5. **Multi-platform** — 11+ chat channels, 21+ LLM providers out of the box
