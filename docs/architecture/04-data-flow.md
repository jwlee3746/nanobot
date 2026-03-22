# 04. Data Flow

## 1. End-to-End Message Lifecycle

```mermaid
flowchart LR
    subgraph Input
        A[User Message]
    end

    subgraph Ingestion
        B[Channel Adapter]
        C[Permission Check<br/>is_allowed]
        D[InboundMessage<br/>construction]
    end

    subgraph Bus
        E[Inbound Queue]
    end

    subgraph Processing
        F[AgentLoop._dispatch]
        G[Session Lookup]
        H[Token Consolidation<br/>Check]
        I[Context Assembly]
        J[LLM Call]
        K{Tool Calls?}
        L[Tool Execution]
        M[Final Response]
    end

    subgraph Persistence
        N[Session Save<br/>JSONL]
        O[Memory Consolidation<br/>Background]
    end

    subgraph Output
        P[Outbound Queue]
        Q[Channel Dispatch]
        R[User Response]
    end

    A --> B --> C -->|allowed| D --> E --> F --> G --> H --> I --> J --> K
    K -->|Yes| L --> J
    K -->|No| M --> N --> O
    M --> P --> Q --> R

    C -->|denied| X[Access Denied<br/>Logged & Dropped]
```

## 2. Context Assembly Pipeline

The `ContextBuilder.build_messages()` method assembles the complete prompt sent to the LLM.

```mermaid
flowchart TB
    subgraph SystemPrompt["System Prompt Assembly"]
        direction TB
        ID["1. Identity Block<br/>Runtime info, workspace path,<br/>platform policy, guidelines"]
        BS["2. Bootstrap Files<br/>AGENTS.md, SOUL.md,<br/>USER.md, TOOLS.md"]
        MEM["3. Memory Context<br/>MEMORY.md long-term facts"]
        ASK["4. Always-On Skills<br/>Skills with always=true"]
        SKL["5. Skills Summary<br/>Available skill list"]
    end

    subgraph Messages["Message List Assembly"]
        direction TB
        SYS["system: {system_prompt}"]
        HIST["...history messages...<br/>(from session, unconsolidated)"]
        USR["user: {runtime_context}<br/>+ {user_message}<br/>+ {media if any}"]
    end

    ID --> BS --> MEM --> ASK --> SKL
    SKL -->|join with ---| SYS
    SYS --> HIST --> USR

    USR -->|"Final messages[]"| LLM["LLM Provider"]
```

### System Prompt Structure

```
# nanobot 🐈                              ← Identity
You are nanobot, a helpful AI assistant.
## Runtime / Workspace / Platform Policy / Guidelines

---

## AGENTS.md                               ← Bootstrap files
{user-customized agent persona}

## SOUL.md
{behavior guidelines}

## USER.md / TOOLS.md

---

# Memory                                   ← Long-term memory
## Long-term Memory
{content of MEMORY.md}

---

# Active Skills                            ← Always-on skills
{loaded SKILL.md content}

---

# Skills                                   ← Available skills catalog
| Skill | Description | Available |
```

## 3. Session Data Model

```mermaid
flowchart TB
    subgraph Session["Session Object"]
        KEY["key: 'telegram:12345'"]
        MSGS["messages: [...]"]
        LC["last_consolidated: 42"]
        META["metadata: {...}"]
    end

    subgraph MessageTypes["Message Types in Session"]
        UM["User Message<br/>{role: 'user', content: '...'}"]
        AM["Assistant Message<br/>{role: 'assistant', content: '...',<br/>tool_calls: [...]}"]
        TM["Tool Result<br/>{role: 'tool', tool_call_id: '...',<br/>name: '...', content: '...'}"]
    end

    subgraph History["get_history() Pipeline"]
        direction LR
        H1["1. Slice from<br/>last_consolidated"]
        H2["2. Take last<br/>max_messages"]
        H3["3. Drop leading<br/>non-user msgs"]
        H4["4. Fix orphan<br/>tool results"]
    end

    subgraph Storage["File Storage"]
        JSONL["sessions/{safe_key}.jsonl<br/>(append-only)"]
    end

    Session --> MessageTypes
    MSGS --> History
    H1 --> H2 --> H3 --> H4
    Session --> JSONL
```

### Session Key Format

```
{channel}:{chat_id}

Examples:
  telegram:12345678
  discord:987654321
  slack:C01ABC123:thread_ts=1234567890.123456
  cli:direct
```

## 4. Memory Consolidation Data Flow

```mermaid
flowchart TB
    subgraph Trigger["Trigger Conditions"]
        T1["Estimated prompt tokens<br/>> context_window_tokens"]
    end

    subgraph Process["Consolidation Process"]
        direction TB
        P1["Pick boundary at user-turn<br/>that removes enough tokens"]
        P2["Extract message chunk<br/>[last_consolidated : boundary]"]
        P3["Format as conversation text"]
        P4["Send to LLM with<br/>current MEMORY.md"]
        P5["LLM calls save_memory tool"]
    end

    subgraph Output["Output"]
        O1["HISTORY.md +=<br/>[YYYY-MM-DD HH:MM] summary"]
        O2["MEMORY.md =<br/>updated facts"]
        O3["session.last_consolidated<br/>= boundary index"]
    end

    subgraph Fallback["Fallback (3 consecutive failures)"]
        F1["Raw archive:<br/>dump messages to HISTORY.md<br/>without LLM summarization"]
    end

    Trigger --> P1 --> P2 --> P3 --> P4 --> P5
    P5 -->|success| O1 & O2 & O3
    P5 -->|failure x3| F1
```

### Token Budget

```
context_window_tokens (e.g. 65,536)
├── System Prompt (~2,000-5,000 tokens)
├── Tool Definitions (~1,000-3,000 tokens)
├── History Messages (variable)
└── Current User Message + Response

Consolidation Target: context_window / 2 (e.g. 32,768)
Max Consolidation Rounds: 5
```

## 5. Tool Execution Data Flow

```mermaid
flowchart TB
    subgraph Request["LLM Tool Call Request"]
        TC["ToolCallRequest<br/>id: 'call_abc'<br/>name: 'web_search'<br/>arguments: {query: '...'}"]
    end

    subgraph Registry["ToolRegistry.execute()"]
        direction TB
        R1["1. Look up tool by name"]
        R2["2. Cast params to schema types"]
        R3["3. Validate params"]
        R4["4. Execute tool"]
        R5["5. Truncate result<br/>(max 16,000 chars)"]
    end

    subgraph Result["Tool Result Message"]
        TR["{role: 'tool',<br/>tool_call_id: 'call_abc',<br/>name: 'web_search',<br/>content: '...'}"]
    end

    TC --> R1 --> R2 --> R3 --> R4 --> R5 --> TR
    TR -->|"Appended to<br/>messages[]"| NEXT["Next LLM Call"]
```

## 6. Provider Selection and Routing

```mermaid
flowchart TB
    subgraph Input["Configuration"]
        MODEL["model: 'anthropic/claude-opus-4-5'"]
        PROV["provider: 'auto'"]
        KEYS["providers.anthropic.api_key: 'sk-...'"]
    end

    subgraph Detection["Auto-Detection (provider='auto')"]
        direction TB
        D1["1. Check model prefix<br/>(anthropic/, openai/, etc.)"]
        D2["2. Check API key prefix<br/>(sk-ant-, gsk_, etc.)"]
        D3["3. Check base URL patterns"]
        D4["4. Check model name keywords<br/>(claude, gpt, deepseek, etc.)"]
    end

    subgraph Routing["Provider Routing"]
        direction TB
        R1{"Provider type?"}
        R2["LiteLLM Provider<br/>(most providers)"]
        R3["Azure OpenAI Provider<br/>(direct API)"]
        R4["Custom Provider<br/>(OpenAI-compatible)"]
        R5["Codex/Copilot Provider<br/>(OAuth)"]
    end

    subgraph Call["LLM API Call"]
        C1["chat_with_retry()"]
        C2["Retry with exponential backoff<br/>delays: 1s, 2s, 4s"]
        C3["Transient error detection<br/>(429, rate limit, timeout, 5xx)"]
    end

    Input --> Detection
    D1 --> D2 --> D3 --> D4
    Detection --> Routing
    R1 -->|litellm| R2
    R1 -->|azure_openai| R3
    R1 -->|custom| R4
    R1 -->|codex/copilot| R5
    Routing --> Call
    C1 --> C2 --> C3
```

## 7. Channel Message Routing

```mermaid
flowchart TB
    subgraph Inbound["Inbound Path"]
        direction LR
        CH_IN["Channel.start()<br/>Listen for events"]
        PERM["Permission Check<br/>is_allowed()"]
        CONV["Convert to<br/>InboundMessage"]
        PUB_IN["bus.publish_inbound()"]
    end

    subgraph Queue["MessageBus"]
        IQ["Inbound Queue"]
        OQ["Outbound Queue"]
    end

    subgraph Outbound["Outbound Path"]
        direction LR
        DISP["ChannelManager<br/>._dispatch_outbound()"]
        FILT["Filter progress<br/>(send_progress,<br/>send_tool_hints)"]
        ROUTE["Route to channel<br/>by msg.channel"]
        SEND["Channel.send()"]
    end

    CH_IN --> PERM --> CONV --> PUB_IN --> IQ
    OQ --> DISP --> FILT --> ROUTE --> SEND
```

### InboundMessage Fields

| Field | Type | Description |
|-------|------|-------------|
| `channel` | `str` | Channel name (telegram, discord, slack, ...) |
| `sender_id` | `str` | User identifier on the platform |
| `chat_id` | `str` | Chat/channel/room identifier |
| `content` | `str` | Message text |
| `timestamp` | `datetime` | Message timestamp |
| `media` | `list[str]` | Media file paths (images, audio) |
| `metadata` | `dict` | Channel-specific data (reply context, etc.) |
| `session_key_override` | `str \| None` | Optional override for thread-scoped sessions |

### OutboundMessage Fields

| Field | Type | Description |
|-------|------|-------------|
| `channel` | `str` | Target channel name |
| `chat_id` | `str` | Target chat identifier |
| `content` | `str` | Response text |
| `reply_to` | `str \| None` | Message ID to reply to |
| `media` | `list[str]` | Media attachments |
| `metadata` | `dict` | Routing metadata (_progress, _tool_hint, render_as) |

## 8. Configuration Loading Pipeline

```mermaid
flowchart TB
    subgraph Sources["Config Sources"]
        DEF["Default Values<br/>(Pydantic defaults)"]
        FILE["config.json<br/>(~/.nanobot/config.json)"]
        ENV["Environment Variables<br/>(NANOBOT_*)"]
        CLI_OPT["CLI Options<br/>(--config, --workspace, etc.)"]
    end

    subgraph Loading["Loading Pipeline"]
        L1["load_config(path)"]
        L2["Read JSON file"]
        L3["Migration<br/>(legacy field names)"]
        L4["Pydantic validation<br/>(camelCase ↔ snake_case)"]
        L5["Apply CLI overrides"]
    end

    subgraph Result["Config Object"]
        CFG["Config<br/>├── agents.defaults<br/>├── providers.*<br/>├── channels.*<br/>├── tools.*<br/>└── gateway.*"]
    end

    DEF --> L4
    FILE --> L2 --> L3 --> L4
    ENV --> L4
    CLI_OPT --> L5
    L4 --> L5 --> CFG
```
