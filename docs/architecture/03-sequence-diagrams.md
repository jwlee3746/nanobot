# 03. Sequence Diagrams

## 1. Gateway Mode Startup

```mermaid
sequenceDiagram
    participant User
    participant CLI as CLI (commands.py)
    participant Cfg as Config Loader
    participant PR as Provider Registry
    participant AL as AgentLoop
    participant CM as ChannelManager
    participant Cron as CronService
    participant HB as HeartbeatService

    User->>CLI: nanobot gateway
    CLI->>Cfg: load_config()
    Cfg-->>CLI: Config object

    CLI->>PR: create_provider(config)
    PR-->>CLI: LLMProvider instance

    CLI->>AL: AgentLoop(bus, provider, workspace, ...)
    Note over AL: Registers default tools<br/>(filesystem, shell, web,<br/>message, spawn, cron)

    CLI->>CM: ChannelManager(config, bus)
    Note over CM: discover_all() via registry<br/>Initialize enabled channels

    CLI->>Cron: CronService(store_path, on_job)
    CLI->>HB: HeartbeatService(workspace, provider, ...)

    par Start all concurrently
        CLI->>AL: run() — main agent loop
        CLI->>CM: start_all() — all channels + dispatcher
        CLI->>Cron: start() — scheduler loop
        CLI->>HB: start() — periodic heartbeat
    end
```

## 2. Message Processing Flow (Gateway Mode)

```mermaid
sequenceDiagram
    participant User
    participant CH as Channel<br/>(e.g. Telegram)
    participant Bus as MessageBus
    participant AL as AgentLoop
    participant CB as ContextBuilder
    participant SS as SessionManager
    participant MC as MemoryConsolidator
    participant LLM as LLM Provider
    participant TR as ToolRegistry
    participant Tool as Tool<br/>(e.g. web_search)

    User->>CH: Send message
    CH->>CH: is_allowed(sender_id)?
    CH->>Bus: publish_inbound(InboundMessage)
    Bus-->>AL: consume_inbound()

    Note over AL: Check slash commands<br/>(/new, /stop, /status, /help)

    AL->>SS: get_or_create(session_key)
    SS-->>AL: Session

    AL->>MC: maybe_consolidate_by_tokens(session)
    Note over MC: If prompt > context_window,<br/>archive old messages

    AL->>SS: get_history()
    SS-->>AL: message history

    AL->>CB: build_messages(history, content, media, ...)
    Note over CB: 1. Build system prompt<br/>   (identity + bootstrap files<br/>    + memory + skills)<br/>2. Append history<br/>3. Append user message<br/>   with runtime context
    CB-->>AL: messages[]

    loop Agent Loop (max 40 iterations)
        AL->>LLM: chat_with_retry(messages, tools)
        LLM-->>AL: LLMResponse

        alt Has tool calls
            AL->>Bus: publish_outbound(progress/hint)
            loop For each tool call
                AL->>TR: execute(tool_name, params)
                TR->>Tool: validate + execute
                Tool-->>TR: result
                TR-->>AL: result (max 16KB)
            end
            Note over AL: Append assistant message<br/>+ tool results to messages[]
        else Final response (no tool calls)
            Note over AL: Break loop
        end
    end

    AL->>SS: save_turn(session, messages)
    AL->>SS: save(session)

    AL->>MC: maybe_consolidate_by_tokens(session)
    Note over MC: Background task

    AL->>Bus: publish_outbound(OutboundMessage)
    Bus-->>CH: consume_outbound()
    CH->>User: Send response
```

## 3. Agent Mode (CLI) Interaction

```mermaid
sequenceDiagram
    participant User
    participant CLI as CLI (agent command)
    participant PT as prompt_toolkit
    participant AL as AgentLoop
    participant Bus as MessageBus

    User->>CLI: nanobot agent
    CLI->>CLI: load_config, create provider
    CLI->>AL: AgentLoop(bus, provider, ...)

    alt Single-turn mode (--message)
        CLI->>AL: process_direct(content)
        AL-->>CLI: OutboundMessage
        CLI->>User: Print response
    else Interactive mode
        loop Until exit command
            CLI->>PT: prompt_async("You: ")
            PT-->>CLI: user input
            CLI->>CLI: Check exit commands

            CLI->>Bus: publish_inbound(InboundMessage)
            Note over CLI: Start thinking spinner

            Bus-->>AL: consume message
            AL->>AL: process_message()

            loop Progress updates
                AL->>Bus: publish_outbound(progress)
                Bus-->>CLI: progress text
                CLI->>User: Print "↳ thinking..."
            end

            AL->>Bus: publish_outbound(response)
            Bus-->>CLI: final response
            CLI->>User: Render markdown response
        end
    end
```

## 4. Memory Consolidation Process

```mermaid
sequenceDiagram
    participant AL as AgentLoop
    participant MC as MemoryConsolidator
    participant MS as MemoryStore
    participant LLM as LLM Provider
    participant FS as File System

    AL->>MC: maybe_consolidate_by_tokens(session)
    MC->>MC: estimate_session_prompt_tokens()

    alt estimated < context_window
        Note over MC: No consolidation needed
    else estimated >= context_window
        Note over MC: Target: context_window / 2

        loop Up to 5 rounds
            MC->>MC: pick_consolidation_boundary(tokens_to_remove)
            Note over MC: Find user-turn boundary<br/>that removes enough tokens

            MC->>MS: consolidate(chunk, provider, model)

            MS->>FS: read MEMORY.md (current facts)

            MS->>LLM: chat_with_retry(consolidation prompt)
            Note over LLM: System: "You are a memory<br/>consolidation agent"<br/>Tool: save_memory<br/>tool_choice: forced

            LLM-->>MS: save_memory(history_entry, memory_update)

            MS->>FS: append to HISTORY.md
            MS->>FS: update MEMORY.md

            MC->>MC: Update session.last_consolidated
            MC->>MC: Re-estimate tokens
        end
    end
```

## 5. Subagent Execution Flow

```mermaid
sequenceDiagram
    participant AL as Main AgentLoop
    participant LLM1 as LLM (Main)
    participant TR as ToolRegistry
    participant SP as SpawnTool
    participant SM as SubagentManager
    participant LLM2 as LLM (Subagent)
    participant Bus as MessageBus

    AL->>LLM1: chat_with_retry(messages, tools)
    LLM1-->>AL: tool_call: spawn(task, label)

    AL->>TR: execute("spawn", {task, label})
    TR->>SP: execute(task, label)
    SP->>SM: spawn(task, label, channel, chat_id)

    SM->>SM: Create asyncio.Task

    SM-->>SP: "Subagent started (id: abc123)"
    SP-->>TR: result
    TR-->>AL: result
    Note over AL: Continues main loop<br/>(non-blocking)

    par Subagent runs independently
        SM->>SM: Build subagent tools<br/>(no message, no spawn)
        SM->>SM: Build focused system prompt

        loop Max 15 iterations
            SM->>LLM2: chat_with_retry(messages, tools)
            LLM2-->>SM: response

            alt Has tool calls
                SM->>SM: Execute tools
            else Final response
                Note over SM: Break
            end
        end

        SM->>Bus: publish_inbound(system message)
        Note over Bus: InboundMessage<br/>channel="system"<br/>sender_id="subagent"
    end

    Bus-->>AL: consume inbound (subagent result)
    AL->>AL: Process as assistant-role message
    AL->>Bus: publish_outbound(summary to user)
```

## 6. MCP Tool Integration

```mermaid
sequenceDiagram
    participant AL as AgentLoop
    participant MCP as MCP Module
    participant Stack as AsyncExitStack
    participant Server as MCP Server<br/>(external process)
    participant TR as ToolRegistry

    AL->>AL: _connect_mcp() [lazy, one-time]

    loop For each configured MCP server
        MCP->>MCP: Determine transport<br/>(stdio / SSE / HTTP)

        alt stdio transport
            MCP->>Server: Launch subprocess
        else SSE transport
            MCP->>Server: Connect via SSE
        else HTTP (streamable-http)
            MCP->>Server: Connect via HTTP
        end

        MCP->>Stack: Register cleanup handler

        MCP->>Server: list_tools()
        Server-->>MCP: tool definitions[]

        loop For each server tool
            MCP->>MCP: Normalize JSON Schema
            MCP->>MCP: Create MCPToolWrapper
            MCP->>TR: register(wrapper)
        end
    end

    Note over TR: MCP tools now available<br/>alongside native tools

    AL->>TR: execute("mcp_tool_name", params)
    TR->>MCP: MCPToolWrapper.execute()
    MCP->>Server: call_tool(name, arguments)
    Server-->>MCP: result
    MCP-->>TR: formatted result
    TR-->>AL: result
```

## 7. Cron Job Execution

```mermaid
sequenceDiagram
    participant Timer as CronService Timer
    participant CS as CronService
    participant Store as CronStore (JSON)
    participant CB as on_job callback
    participant AL as AgentLoop
    participant Bus as MessageBus
    participant CH as Channel

    Timer->>CS: _tick()
    CS->>Store: Load jobs

    loop For each active job
        CS->>CS: Check if next_run_ms <= now

        alt Job is due
            CS->>CS: Mark state = running
            CS->>CB: on_job(job)

            CB->>AL: process_direct(job.payload.message)
            AL-->>CB: response

            CS->>CS: Compute next_run_ms
            CS->>CS: Record run in history
            CS->>Store: Save updated store

            alt Job has deliver_to channel
                CS->>Bus: publish_outbound(result)
                Bus-->>CH: Send to channel
            end

            alt One-shot "at" schedule
                CS->>CS: Mark state = completed
            end
        end
    end
```

## 8. Heartbeat Service Flow

```mermaid
sequenceDiagram
    participant Timer as HeartbeatService Timer
    participant HB as HeartbeatService
    participant FS as File System
    participant LLM as LLM Provider
    participant Eval as Evaluator
    participant CB as on_execute callback
    participant Notify as on_notify callback

    loop Every interval_s (default 30 min)
        Timer->>HB: _tick()
        HB->>FS: Read HEARTBEAT.md

        alt File missing or empty
            Note over HB: Skip
        else Has content
            HB->>LLM: Phase 1: _decide(content)
            Note over LLM: System: "You are a heartbeat agent"<br/>Tool: heartbeat(action, tasks)<br/>action: "skip" | "run"

            LLM-->>HB: {action, tasks}

            alt action == "skip"
                Note over HB: Nothing to do
            else action == "run"
                HB->>CB: on_execute(tasks)
                CB-->>HB: response

                HB->>Eval: evaluate_response(response, tasks)
                Note over Eval: Determine if result<br/>warrants notification

                alt should_notify
                    HB->>Notify: on_notify(response)
                else
                    Note over HB: Silenced
                end
            end
        end
    end
```
