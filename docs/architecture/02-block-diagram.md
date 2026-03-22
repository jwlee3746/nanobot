# 02. Block Diagram

## System Architecture Overview

```mermaid
graph TB
    subgraph User["User Layer"]
        CLI["CLI<br/>(nanobot agent)"]
        TG["Telegram"]
        DC["Discord"]
        SL["Slack"]
        WA["WhatsApp"]
        FS["Feishu"]
        DT["DingTalk"]
        OC["Other Channels<br/>(QQ, WeCom, Matrix,<br/>Email, MoChat)"]
    end

    subgraph Gateway["Gateway Layer"]
        CM["ChannelManager"]
        CR["ChannelRegistry<br/>(pkgutil + entry_points)"]
    end

    subgraph Bus["Message Bus Layer"]
        IQ["Inbound Queue<br/>(asyncio.Queue)"]
        OQ["Outbound Queue<br/>(asyncio.Queue)"]
    end

    subgraph Core["Agent Core"]
        AL["AgentLoop"]
        CB["ContextBuilder"]
        TR["ToolRegistry"]
        SM["SubagentManager"]
        MC["MemoryConsolidator"]
    end

    subgraph Tools["Tool Layer"]
        FT["Filesystem Tools<br/>(read, write, edit, ls)"]
        ST["Shell Tool<br/>(exec)"]
        WT["Web Tools<br/>(search, fetch)"]
        MT["Message Tool"]
        SPT["Spawn Tool"]
        CT["Cron Tool"]
        MCP["MCP Tools<br/>(external servers)"]
    end

    subgraph LLM["LLM Provider Layer"]
        PR["Provider Registry"]
        AN["Anthropic"]
        OA["OpenAI"]
        DS["DeepSeek"]
        GM["Gemini"]
        OL["Ollama"]
        OT["Other Providers<br/>(21+ total)"]
    end

    subgraph Persistence["Persistence Layer"]
        SS["SessionManager<br/>(JSONL)"]
        MS["MemoryStore<br/>(MEMORY.md)"]
        HS["History Store<br/>(HISTORY.md)"]
        CS["CronStore<br/>(JSON)"]
        CF["Config<br/>(config.json)"]
    end

    subgraph Services["Background Services"]
        CRS["CronService"]
        HBS["HeartbeatService"]
    end

    %% User → Gateway
    TG & DC & SL & WA & FS & DT & OC --> CM
    CM --> CR

    %% CLI direct
    CLI --> IQ

    %% Gateway ↔ Bus
    CM -->|"publish_inbound()"| IQ
    OQ -->|"consume_outbound()"| CM
    CM -->|"channel.send()"| TG & DC & SL & WA & FS & DT & OC

    %% Bus ↔ Core
    IQ -->|"consume_inbound()"| AL
    AL -->|"publish_outbound()"| OQ

    %% Core internal
    AL --> CB
    AL --> TR
    AL --> SM
    AL --> MC

    %% Tools
    TR --> FT & ST & WT & MT & SPT & CT & MCP

    %% LLM
    AL -->|"chat_with_retry()"| PR
    PR --> AN & OA & DS & GM & OL & OT

    %% Persistence
    AL --> SS
    MC --> MS & HS
    CRS --> CS

    %% Services → Bus
    CRS -->|"scheduled messages"| IQ
    HBS -->|"heartbeat triggers"| IQ

    %% Subagent
    SM -->|"spawn background tasks"| PR
    SM -->|"announce results"| IQ
```

## Component Relationships (Simplified)

```mermaid
graph LR
    subgraph Input
        CH[Channels]
        CLI[CLI]
    end

    subgraph Processing
        BUS[MessageBus]
        AGENT[AgentLoop]
        LLM[LLM Provider]
    end

    subgraph Output
        RESP[Response]
        TOOLS[Tool Execution]
        MEM[Memory Update]
    end

    CH -->|InboundMessage| BUS
    CLI -->|InboundMessage| BUS
    BUS -->|consume| AGENT
    AGENT -->|chat_with_retry| LLM
    LLM -->|LLMResponse| AGENT
    AGENT -->|OutboundMessage| BUS
    BUS --> RESP
    AGENT --> TOOLS
    AGENT --> MEM
    TOOLS -->|result| AGENT
```

## Tool System Architecture

```mermaid
classDiagram
    class Tool {
        <<abstract>>
        +name: str
        +description: str
        +parameters: dict
        +execute(**kwargs) Any
        +to_schema() dict
        +validate_params(params) list
        +cast_params(params) dict
    }

    class ToolRegistry {
        -_tools: dict[str, Tool]
        +register(tool)
        +unregister(name)
        +get(name) Tool
        +execute(name, params) Any
        +get_definitions() list[dict]
    }

    class ReadFileTool { }
    class WriteFileTool { }
    class EditFileTool { }
    class ListDirTool { }
    class ExecTool { }
    class WebSearchTool { }
    class WebFetchTool { }
    class MessageTool { }
    class SpawnTool { }
    class CronTool { }
    class MCPToolWrapper { }

    Tool <|-- ReadFileTool
    Tool <|-- WriteFileTool
    Tool <|-- EditFileTool
    Tool <|-- ListDirTool
    Tool <|-- ExecTool
    Tool <|-- WebSearchTool
    Tool <|-- WebFetchTool
    Tool <|-- MessageTool
    Tool <|-- SpawnTool
    Tool <|-- CronTool
    Tool <|-- MCPToolWrapper

    ToolRegistry o-- Tool : manages
```

## Channel System Architecture

```mermaid
classDiagram
    class BaseChannel {
        <<abstract>>
        +name: str
        +display_name: str
        +config: Any
        +bus: MessageBus
        +start()
        +stop()
        +send(msg: OutboundMessage)
        +is_allowed(sender_id) bool
        #_handle_message(sender_id, chat_id, content, ...)
        +transcribe_audio(file_path) str
    }

    class ChannelManager {
        +channels: dict[str, BaseChannel]
        +start_all()
        +stop_all()
        -_dispatch_outbound()
        -_init_channels()
    }

    class ChannelRegistry {
        +discover_all() dict
    }

    class TelegramChannel { }
    class DiscordChannel { }
    class SlackChannel { }
    class WhatsAppChannel { }
    class FeishuChannel { }
    class DingTalkChannel { }
    class QQChannel { }
    class WeComChannel { }
    class MatrixChannel { }
    class EmailChannel { }
    class MoChatChannel { }

    BaseChannel <|-- TelegramChannel
    BaseChannel <|-- DiscordChannel
    BaseChannel <|-- SlackChannel
    BaseChannel <|-- WhatsAppChannel
    BaseChannel <|-- FeishuChannel
    BaseChannel <|-- DingTalkChannel
    BaseChannel <|-- QQChannel
    BaseChannel <|-- WeComChannel
    BaseChannel <|-- MatrixChannel
    BaseChannel <|-- EmailChannel
    BaseChannel <|-- MoChatChannel

    ChannelManager o-- BaseChannel : manages
    ChannelManager ..> ChannelRegistry : uses
```

## Provider System Architecture

```mermaid
classDiagram
    class LLMProvider {
        <<abstract>>
        +chat(messages, tools, model, ...) LLMResponse
        +chat_with_retry(messages, tools, ...) LLMResponse
        +get_default_model() str
        -_is_transient_error(error) bool
    }

    class LLMResponse {
        +content: str | None
        +tool_calls: list[ToolCallRequest]
        +finish_reason: str
        +usage: dict
        +reasoning_content: str | None
        +thinking_blocks: list[dict] | None
        +has_tool_calls: bool
    }

    class ToolCallRequest {
        +id: str
        +name: str
        +arguments: dict
        +to_openai_tool_call() dict
    }

    class GenerationSettings {
        +temperature: float
        +max_tokens: int
        +reasoning_effort: str | None
    }

    class LiteLLMProvider { }
    class AzureOpenAIProvider { }
    class CustomProvider { }
    class OpenAICodexProvider { }

    class ProviderRegistry {
        +PROVIDERS: dict[str, ProviderSpec]
        +detect_provider(model, api_key, base_url) str
    }

    LLMProvider <|-- LiteLLMProvider
    LLMProvider <|-- AzureOpenAIProvider
    LLMProvider <|-- CustomProvider
    LLMProvider <|-- OpenAICodexProvider

    LLMProvider ..> LLMResponse : returns
    LLMResponse o-- ToolCallRequest
    LLMProvider ..> GenerationSettings : uses
    ProviderRegistry ..> LLMProvider : creates
```

## Memory System Architecture

```mermaid
graph TB
    subgraph MemorySystem["Memory System"]
        MS["MemoryStore"]
        MC["MemoryConsolidator"]
    end

    subgraph Storage["File Storage"]
        MEM["MEMORY.md<br/>(long-term facts)"]
        HIS["HISTORY.md<br/>(grep-searchable log)"]
    end

    subgraph Context["Context Integration"]
        CB["ContextBuilder"]
        SP["System Prompt"]
    end

    MC -->|"consolidate()"| MS
    MS -->|"write_long_term()"| MEM
    MS -->|"append_history()"| HIS
    MS -->|"get_memory_context()"| CB
    CB -->|"includes memory"| SP

    MC -->|"estimate tokens"| TC["Token Counter"]
    MC -->|"pick_consolidation_boundary()"| BD["Boundary Detection"]
    BD -->|"user-turn aligned"| MC

    MC -->|"LLM call with<br/>save_memory tool"| LLM["LLM Provider"]
    LLM -->|"history_entry +<br/>memory_update"| MC
```

## Configuration Schema

```mermaid
classDiagram
    class Config {
        +agents: AgentsConfig
        +providers: ProvidersConfig
        +channels: ChannelsConfig
        +tools: ToolsConfig
        +gateway: GatewayConfig
    }

    class AgentsConfig {
        +defaults: AgentDefaults
    }

    class AgentDefaults {
        +workspace: str
        +model: str
        +provider: str
        +max_tokens: int
        +context_window_tokens: int
        +temperature: float
        +max_tool_iterations: int
        +reasoning_effort: str | None
    }

    class ProvidersConfig {
        +anthropic: ProviderConfig
        +openai: ProviderConfig
        +deepseek: ProviderConfig
        +gemini: ProviderConfig
        +ollama: ProviderConfig
        ... (21 providers)
    }

    class ProviderConfig {
        +api_key: str
        +api_base: str | None
        +extra_headers: dict | None
    }

    class ChannelsConfig {
        +send_progress: bool
        +send_tool_hints: bool
        ... (extra fields per channel)
    }

    class ToolsConfig {
        +web: WebToolsConfig
        +exec: ExecToolConfig
        +mcp_servers: dict
    }

    class GatewayConfig {
        +host: str
        +port: int
        +heartbeat: HeartbeatConfig
    }

    Config *-- AgentsConfig
    Config *-- ProvidersConfig
    Config *-- ChannelsConfig
    Config *-- ToolsConfig
    Config *-- GatewayConfig
    AgentsConfig *-- AgentDefaults
    ProvidersConfig *-- ProviderConfig
    GatewayConfig *-- HeartbeatConfig
```
