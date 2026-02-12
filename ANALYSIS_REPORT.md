# MimiClaw Project Analysis Report

**Generated:** 2025-01-27  
**Project:** MimiClaw - ESP32-S3 AI Assistant  
**Language:** C (ESP-IDF v5.5+)  
**Target Platform:** ESP32-S3 (16 MB flash, 8 MB PSRAM)

---

## Executive Summary

MimiClaw is an innovative embedded AI assistant that runs entirely on a $5 ESP32-S3 microcontroller. It implements a complete AI agent architecture using pure C, FreeRTOS, and the Anthropic Claude API, without requiring Linux, Node.js, or any server infrastructure. The system provides Telegram bot integration, WebSocket gateway, local memory persistence, and tool-use capabilities (ReAct pattern) in a resource-constrained environment.

**Key Achievement:** Full AI agent loop with tool execution, memory management, and multi-channel communication on a microcontroller with only 8 MB PSRAM and 16 MB flash.

---

## 1. Project Structure

### 1.1 Directory Organization

```
mimiclaw/
├── main/                    # Core application code
│   ├── agent/              # Agent loop and context building
│   ├── bus/                # Message bus (FreeRTOS queues)
│   ├── cli/                # Serial CLI interface
│   ├── gateway/            # WebSocket server
│   ├── llm/                # Anthropic API client
│   ├── memory/             # Memory store and session management
│   ├── ota/                # Over-the-air updates
│   ├── proxy/              # HTTP CONNECT proxy support
│   ├── telegram/           # Telegram bot integration
│   ├── tools/              # Tool registry and implementations
│   ├── wifi/               # WiFi management
│   ├── mimi.c              # Application entry point
│   ├── mimi_config.h        # Configuration constants
│   └── mimi_secrets.h.example  # Secrets template
├── docs/                    # Architecture and TODO documentation
├── spiffs_data/            # SPIFFS filesystem data
│   ├── config/             # SOUL.md, USER.md
│   └── memory/             # MEMORY.md
├── CMakeLists.txt          # Build configuration
├── partitions.csv          # Flash partition layout
└── sdkconfig.defaults      # ESP-IDF defaults
```

### 1.2 Module Breakdown

| Module | Files | Purpose |
|--------|-------|---------|
| **Agent** | `agent_loop.c/h`, `context_builder.c/h` | ReAct loop, system prompt construction |
| **Bus** | `message_bus.c/h` | FreeRTOS queue-based message routing |
| **LLM** | `llm_proxy.c/h` | Anthropic Messages API client |
| **Memory** | `memory_store.c/h`, `session_mgr.c/h` | SPIFFS-based persistence |
| **Tools** | `tool_registry.c/h`, `tool_*.c/h` | Tool registration and execution |
| **Telegram** | `telegram_bot.c/h` | Long polling bot implementation |
| **Gateway** | `ws_server.c/h` | WebSocket server (port 18789) |
| **CLI** | `serial_cli.c/h` | Serial console REPL |
| **Proxy** | `http_proxy.c/h` | HTTP CONNECT tunnel support |
| **WiFi** | `wifi_manager.c/h` | WiFi STA connection management |
| **OTA** | `ota_manager.c/h` | Firmware update over WiFi |

---

## 2. Architecture Overview

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────┐
│           ESP32-S3 (Dual Core)                  │
│                                                 │
│  Core 0 (I/O):              Core 1 (Agent):     │
│  ┌─────────────┐           ┌──────────────┐   │
│  │ Telegram    │           │ Agent Loop    │   │
│  │ Poller      │──────────▶│ (ReAct)      │   │
│  └─────────────┘           │              │   │
│  ┌─────────────┐           │  - Context   │   │
│  │ WebSocket   │──────────▶│  - LLM Call  │   │
│  │ Server      │           │  - Tool Exec │   │
│  └─────────────┘           └──────┬───────┘   │
│  ┌─────────────┐                  │           │
│  │ Outbound    │◀─────────────────┘           │
│  │ Dispatch    │                              │
│  └─────────────┘                              │
│  ┌─────────────┐                              │
│  │ Serial CLI  │                              │
│  └─────────────┘                              │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │ SPIFFS (12 MB)                           │  │
│  │ - config/ (SOUL.md, USER.md)            │  │
│  │ - memory/ (MEMORY.md, YYYY-MM-DD.md)    │  │
│  │ - sessions/ (tg_<chat_id>.jsonl)        │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
   Telegram API         Anthropic API
   Brave Search API
```

### 2.2 Data Flow

1. **Input:** User sends message via Telegram or WebSocket
2. **Ingestion:** Channel poller receives message, wraps in `mimi_msg_t`
3. **Queue:** Message pushed to inbound FreeRTOS queue
4. **Processing:** Agent loop (Core 1) processes:
   - Loads session history from SPIFFS (JSONL format)
   - Builds system prompt (SOUL.md + USER.md + MEMORY.md + recent notes)
   - ReAct loop (max 10 iterations):
     - Calls Claude API with tools
     - Parses response (text + tool_use blocks)
     - Executes tools (web_search, get_time, file ops)
     - Appends tool results to conversation
     - Repeats until `stop_reason == "end_turn"`
   - Saves conversation to session file
5. **Output:** Response pushed to outbound queue
6. **Dispatch:** Outbound task (Core 0) routes to appropriate channel
7. **Delivery:** User receives reply

### 2.3 FreeRTOS Task Layout

| Task | Core | Priority | Stack | Description |
|------|------|----------|-------|-------------|
| `tg_poll` | 0 | 5 | 12 KB | Telegram long polling (30s timeout) |
| `agent_loop` | 1 | 6 | 12 KB | Message processing + Claude API call |
| `outbound` | 0 | 5 | 8 KB | Route responses to Telegram / WS |
| `serial_cli` | 0 | 3 | 4 KB | USB serial console REPL |
| `httpd` (internal) | 0 | 5 | — | WebSocket server (esp_http_server) |
| `wifi_event` (IDF) | 0 | 8 | — | WiFi event handling (ESP-IDF) |

**Core Allocation Strategy:** Core 0 handles I/O (network, serial, WiFi). Core 1 is dedicated to CPU-bound agent processing (JSON building, HTTPS calls).

---

## 3. Core Components & Functions

### 3.1 Agent Loop (`agent/agent_loop.c`)

**Purpose:** Implements the ReAct (Reasoning + Acting) agent pattern with tool use.

**Key Functions:**
- `agent_loop_init()`: Initialize agent loop
- `agent_loop_start()`: Launch agent task on Core 1
- `agent_loop_task()`: Main processing loop

**ReAct Loop Implementation:**
```c
while (iteration < MIMI_AGENT_MAX_TOOL_ITER) {
    1. Call llm_chat_tools() with system prompt + messages + tools
    2. Parse response → text blocks + tool_use blocks
    3. If tool_use:
       - Execute each tool (tool_registry_execute)
       - Append assistant message with tool_use blocks
       - Append user message with tool_result blocks
       - Continue loop
    4. If end_turn: break with final text
}
```

**Features:**
- Maximum 10 tool iterations per message
- Working indicators sent before each API call
- PSRAM allocation for large buffers (system prompt, history, tool output)
- Error handling with fallback messages

**Limitations:**
- No memory write via tool use (agent cannot persist memories autonomously)
- Fixed iteration limit (10) - no adaptive stopping
- No streaming token support (full response buffered)

### 3.2 LLM Proxy (`llm/llm_proxy.c`)

**Purpose:** Anthropic Messages API client with tool use support.

**Key Functions:**
- `llm_proxy_init()`: Load API key and model from secrets/NVS
- `llm_chat_tools()`: Send request with tools, parse tool_use response
- `llm_set_api_key()`: Runtime API key update (NVS)
- `llm_set_model()`: Runtime model selection (NVS)

**API Integration:**
- Endpoint: `https://api.anthropic.com/v1/messages`
- Protocol: Anthropic Messages API (non-streaming)
- Authentication: `x-api-key` header
- Request format: `{model, max_tokens, system, messages, tools}`
- Response parsing: Extracts text blocks and `tool_use` blocks

**Features:**
- HTTP CONNECT proxy support via `proxy/http_proxy.c`
- Non-streaming JSON response (buffers full response)
- Tool use protocol support (Anthropic-native)
- Error handling with status code checking

**Limitations:**
- Hardcoded to Anthropic API (no multi-provider support)
- No streaming support (buffers entire response)
- Fixed max_tokens (4096)
- No retry logic for transient failures

### 3.3 Memory Store (`memory/memory_store.c`)

**Purpose:** Long-term and daily memory persistence on SPIFFS.

**Key Functions:**
- `memory_read_long_term()`: Read `MEMORY.md`
- `memory_write_long_term()`: Write `MEMORY.md`
- `memory_append_today()`: Append to `YYYY-MM-DD.md`
- `memory_read_recent()`: Read last N days of daily notes

**Storage Layout:**
```
/spiffs/
├── config/
│   ├── SOUL.md          # AI personality
│   └── USER.md          # User profile
├── memory/
│   ├── MEMORY.md        # Long-term memory
│   └── 2026-02-05.md    # Daily notes (one per day)
└── sessions/
    └── tg_12345.jsonl   # Chat history (JSONL format)
```

**Features:**
- Plain text markdown files (human-readable)
- Daily notes automatically dated
- Recent memory lookback (default: 3 days, configurable)

**Limitations:**
- No memory write via tool use (only CLI can write)
- Fixed 3-day lookback (not configurable per-request)
- No memory compaction or summarization
- No automatic memory cleanup

### 3.4 Session Manager (`memory/session_mgr.c`)

**Purpose:** Per-chat conversation history management.

**Key Functions:**
- `session_append()`: Append message to JSONL file
- `session_get_history_json()`: Load last N messages as JSON array
- `session_clear()`: Delete session file
- `session_list()`: List all session files

**Format:**
```jsonl
{"role":"user","content":"Hello","ts":1738764800}
{"role":"assistant","content":"Hi there!","ts":1738764802}
```

**Features:**
- Ring buffer behavior (last 20 messages by default)
- JSONL format (one JSON object per line)
- Automatic timestamping
- Per-chat isolation (one file per chat_id)

**Limitations:**
- No session metadata (created_at, updated_at)
- Fixed max messages (20) - no adaptive truncation
- No conversation summarization
- No session expiration/cleanup

### 3.5 Tool Registry (`tools/tool_registry.c`)

**Purpose:** Tool registration, JSON schema generation, and execution dispatch.

**Key Functions:**
- `tool_registry_init()`: Register all built-in tools
- `tool_registry_get_tools_json()`: Get tools array for API request
- `tool_registry_execute()`: Execute tool by name

**Registered Tools:**
1. **web_search** (`tool_web_search.c`): Brave Search API integration
2. **get_current_time** (`tool_get_time.c`): Fetch time via HTTP, set system clock
3. **read_file** (`tool_files.c`): Read file from SPIFFS
4. **write_file** (`tool_files.c`): Write file to SPIFFS
5. **edit_file** (`tool_files.c`): Find and replace in file
6. **list_dir** (`tool_files.c`): List SPIFFS directory

**Tool Schema:**
Each tool provides:
- `name`: Tool identifier
- `description`: Natural language description for LLM
- `input_schema_json`: JSON Schema for input validation
- `execute()`: Implementation function

**Features:**
- Path validation (must start with `/spiffs/`, no `..` traversal)
- JSON Schema generation for Anthropic API
- Error handling with descriptive messages
- File size limits (32 KB max for file operations)

**Limitations:**
- No memory write tools (agent cannot persist memories)
- No message sending tool (agent cannot send messages to other chats)
- Limited file operations (no append, no directory creation)
- No tool result caching

### 3.6 Context Builder (`agent/context_builder.c`)

**Purpose:** Construct system prompt from bootstrap files and memory.

**Key Functions:**
- `context_build_system_prompt()`: Build complete system prompt

**System Prompt Structure:**
```
1. SOUL.md (personality/behavior)
2. USER.md (user profile)
3. MEMORY.md (long-term memory)
4. Recent daily notes (last 3 days)
5. Tool usage guidance
```

**Features:**
- Automatic file loading from SPIFFS
- Tool documentation injection
- Memory context integration
- Configurable buffer size (16 KB default)

**Limitations:**
- Only loads SOUL.md and USER.md (missing AGENTS.md, TOOLS.md, IDENTITY.md)
- Fixed 3-day lookback (not configurable)
- No dynamic prompt optimization
- No token counting/budgeting

### 3.7 Message Bus (`bus/message_bus.c`)

**Purpose:** FreeRTOS queue-based message routing between components.

**Key Functions:**
- `message_bus_init()`: Create inbound and outbound queues
- `message_bus_push_inbound()`: Push message to agent loop
- `message_bus_pop_inbound()`: Pop message (blocking)
- `message_bus_push_outbound()`: Push response to dispatch
- `message_bus_pop_outbound()`: Pop response (blocking)

**Message Structure:**
```c
typedef struct {
    char channel[16];    // "telegram", "websocket", "cli"
    char chat_id[32];    // Telegram chat ID or WS client ID
    char *content;       // Heap-allocated text (ownership transferred)
} mimi_msg_t;
```

**Features:**
- Two queues: inbound (channels → agent) and outbound (agent → channels)
- Queue depth: 8 messages each
- Ownership transfer (caller must free content after pop)
- Blocking operations with timeout

**Limitations:**
- Fixed queue depth (8) - no dynamic sizing
- No message priority
- No message metadata (media, reply_to)
- Hardcoded dispatch (if-else, not subscription-based)

### 3.8 Telegram Bot (`telegram/telegram_bot.c`)

**Purpose:** Telegram Bot API integration via long polling.

**Key Functions:**
- `telegram_bot_init()`: Load bot token from secrets/NVS
- `telegram_bot_start()`: Launch polling task
- `telegram_send_message()`: Send message (auto-splits >4096 chars)
- `telegram_set_token()`: Runtime token update (NVS)

**Features:**
- Long polling (30s timeout)
- Message splitting for long responses
- Markdown support (with fallback to plain text)
- HTTP CONNECT proxy support

**Limitations:**
- No media handling (photos, voice, files ignored)
- No `/start` command handling
- No user allowlist (anyone can message bot)
- Basic Markdown (no HTML conversion, special chars can fail)
- No reply-to support

### 3.9 WebSocket Gateway (`gateway/ws_server.c`)

**Purpose:** WebSocket server for LAN access (port 18789).

**Key Functions:**
- `ws_server_start()`: Start HTTP server with WebSocket upgrade
- `ws_server_send()`: Send message to WebSocket client

**Protocol:**
```json
// Client → Server
{"type": "message", "content": "Hello", "chat_id": "ws_client1"}

// Server → Client
{"type": "response", "content": "Hi there!", "chat_id": "ws_client1"}
```

**Features:**
- Max 4 concurrent clients
- Auto-assigned client IDs (`ws_<fd>`)
- JSON message format
- HTTP server integration (esp_http_server)

**Limitations:**
- No streaming token push
- Basic protocol (no authentication)
- No reconnection handling
- Fixed max clients (4)

### 3.10 Serial CLI (`cli/serial_cli.c`)

**Purpose:** USB serial console for debugging and maintenance.

**Key Commands:**
- `wifi_status`: Show WiFi connection status
- `memory_read`: Print MEMORY.md contents
- `memory_write <CONTENT>`: Overwrite MEMORY.md
- `session_list`: List all session files
- `session_clear <CHAT_ID>`: Delete session file
- `heap_info`: Show free memory (internal + PSRAM)
- `restart`: Reboot device
- `wifi_set <SSID> <PASS>`: Change WiFi (NVS)
- `set_tg_token <TOKEN>`: Change Telegram token (NVS)
- `set_api_key <KEY>`: Change API key (NVS)
- `set_model <MODEL>`: Change LLM model (NVS)
- `set_proxy <HOST> <PORT>`: Set HTTP proxy (NVS)
- `clear_proxy`: Remove proxy (NVS)
- `set_search_key <KEY>`: Set Brave Search key (NVS)
- `config_show`: Show all config (masked)
- `config_reset`: Clear NVS, revert to build-time defaults

**Features:**
- Works without WiFi (available immediately)
- Runtime configuration (NVS override)
- Debug and maintenance commands
- esp_console REPL

**Limitations:**
- No command history
- No tab completion
- Basic error messages

---

## 4. Configuration System

### 4.1 Two-Layer Configuration

**Layer 1: Build-Time Secrets (`mimi_secrets.h`)**
- Highest priority
- Compiled into firmware
- Requires rebuild to change
- Git-ignored file

**Layer 2: Runtime NVS Override**
- Lower priority (overridden by build-time)
- Stored in NVS flash
- Can be changed via CLI without rebuild
- Persists across reboots

### 4.2 Configuration Parameters

| Parameter | Build-Time | Runtime | Description |
|-----------|------------|---------|-------------|
| `MIMI_SECRET_WIFI_SSID` | ✓ | ✓ | WiFi network name |
| `MIMI_SECRET_WIFI_PASS` | ✓ | ✓ | WiFi password |
| `MIMI_SECRET_TG_TOKEN` | ✓ | ✓ | Telegram bot token |
| `MIMI_SECRET_API_KEY` | ✓ | ✓ | Anthropic API key |
| `MIMI_SECRET_MODEL` | ✓ | ✓ | LLM model ID |
| `MIMI_SECRET_PROXY_HOST` | ✓ | ✓ | HTTP proxy host |
| `MIMI_SECRET_PROXY_PORT` | ✓ | ✓ | HTTP proxy port |
| `MIMI_SECRET_SEARCH_KEY` | ✓ | ✓ | Brave Search API key |

### 4.3 Flash Partition Layout

```
Offset      Size      Name        Purpose
─────────────────────────────────────────────
0x009000    24 KB     nvs         ESP-IDF internal use
0x00F000     8 KB     otadata     OTA boot state
0x011000     4 KB     phy_init    WiFi PHY calibration
0x020000     2 MB     ota_0       Firmware slot A
0x220000     2 MB     ota_1       Firmware slot B
0x420000    12 MB     spiffs      Markdown memory, sessions, config
0xFF0000    64 KB     coredump    Crash dump storage
```

**Total:** 16 MB flash

---

## 5. Memory Budget

| Purpose | Location | Size |
|---------|----------|------|
| FreeRTOS task stacks | Internal SRAM | ~40 KB |
| WiFi buffers | Internal SRAM | ~30 KB |
| TLS connections x2 | PSRAM | ~120 KB |
| JSON parse buffers | PSRAM | ~32 KB |
| Session history cache | PSRAM | ~32 KB |
| System prompt buffer | PSRAM | ~16 KB |
| LLM response stream buffer | PSRAM | ~32 KB |
| Tool output buffer | PSRAM | ~8 KB |
| **Remaining available** | **PSRAM** | **~7.7 MB** |

**Large buffers (32 KB+) are allocated from PSRAM via `heap_caps_calloc(1, size, MALLOC_CAP_SPIRAM)`.**

---

## 6. Key Features

### 6.1 Implemented Features

✅ **ReAct Agent Loop**
- Multi-turn tool use (max 10 iterations)
- Anthropic tool use protocol
- Automatic tool execution and result injection

✅ **Memory Persistence**
- Long-term memory (MEMORY.md)
- Daily notes (YYYY-MM-DD.md)
- Session history (JSONL per chat)

✅ **Tool System**
- Tool registry with JSON Schema
- 6 built-in tools (web_search, get_time, file ops)
- Path validation and security

✅ **Multi-Channel Support**
- Telegram bot (long polling)
- WebSocket gateway (port 18789)
- Serial CLI

✅ **HTTP Proxy Support**
- CONNECT tunnel for restricted networks
- Works with Telegram, Claude API, Brave Search

✅ **OTA Updates**
- Over-the-air firmware updates
- Dual slot support (ota_0, ota_1)

✅ **Runtime Configuration**
- NVS-based config override
- CLI commands for all settings
- No rebuild required for config changes

✅ **Dual-Core Architecture**
- Core 0: I/O (network, serial)
- Core 1: Agent processing (CPU-bound)

### 6.2 Missing Features (from TODO.md)

❌ **Memory Write via Tool Use**
- Agent cannot autonomously persist memories
- Only CLI can write MEMORY.md

❌ **More Built-in Tools**
- Missing: `message` (send to other chats)
- File tools limited (no append, no dir creation)

❌ **Telegram Enhancements**
- No media handling (photos, voice, files)
- No `/start` command
- No user allowlist (security risk)
- Basic Markdown (no HTML conversion)

❌ **Bootstrap File Completion**
- Only SOUL.md and USER.md loaded
- Missing: AGENTS.md, TOOLS.md, IDENTITY.md

❌ **Subagent / Background Tasks**
- No subagent spawning
- No background task support

❌ **Cron / Heartbeat**
- No scheduled tasks
- No periodic memory checks

❌ **Multi-LLM Provider**
- Hardcoded to Anthropic API
- No OpenAI/Gemini/etc. support

---

## 7. Limitations & Constraints

### 7.1 Hardware Constraints

**Memory:**
- Limited PSRAM (8 MB total, ~7.7 MB available)
- Internal SRAM (~40 KB for stacks)
- Large buffers must use PSRAM

**Flash:**
- 16 MB total
- 12 MB for SPIFFS (memory/sessions)
- 2 MB per OTA slot
- No external storage

**CPU:**
- Dual-core Xtensa LX7 (240 MHz)
- No hardware floating point
- Limited computational power

**Network:**
- WiFi only (no Ethernet)
- Single WiFi connection
- Limited concurrent connections

### 7.2 Software Constraints

**ESP-IDF:**
- Requires ESP-IDF v5.5+
- FreeRTOS-based
- Limited standard library support

**Language:**
- Pure C (no C++ features)
- Manual memory management
- No exceptions or RAII

**API Limitations:**
- Anthropic API only (no multi-provider)
- Non-streaming responses (full buffering)
- Fixed max_tokens (4096)
- No retry logic

**Storage:**
- SPIFFS filesystem (flat, no real directories)
- File size limits (32 KB for file operations)
- No database support

### 7.3 Functional Limitations

**Agent:**
- Fixed iteration limit (10) - no adaptive stopping
- No memory write via tools
- No conversation summarization
- No context window management

**Memory:**
- Fixed 3-day lookback (not configurable)
- No memory compaction
- No automatic cleanup
- No memory prioritization

**Session:**
- Fixed max messages (20) - no adaptive truncation
- No session expiration
- No conversation summarization
- No metadata storage

**Tools:**
- Limited file operations
- No tool result caching
- No async tool execution
- No tool chaining

**Telegram:**
- No media support
- No user authentication
- Basic Markdown (no HTML)
- No reply-to support

**WebSocket:**
- No streaming tokens
- Fixed max clients (4)
- Basic protocol (no auth)
- No reconnection handling

### 7.4 Security Limitations

**Authentication:**
- No Telegram user allowlist (anyone can message bot)
- No WebSocket authentication
- No API key rotation

**Data Protection:**
- Secrets stored in plaintext (NVS)
- No encryption for SPIFFS
- No secure boot (mentioned but not verified)

**Network:**
- HTTP proxy support (but no authentication)
- No TLS certificate pinning
- No rate limiting

### 7.5 Scalability Limitations

**Concurrent Users:**
- Single agent loop (sequential processing)
- Queue depth: 8 messages
- Max 4 WebSocket clients

**Storage:**
- Fixed SPIFFS size (12 MB)
- No external storage
- No cloud sync

**Performance:**
- Single-threaded agent processing
- No request queuing/prioritization
- No caching of API responses

---

## 8. Code Quality & Design Patterns

### 8.1 Strengths

✅ **Modular Design**
- Clear separation of concerns
- Well-defined module interfaces
- Minimal coupling between modules

✅ **Resource Management**
- PSRAM allocation for large buffers
- Proper memory cleanup (free on pop)
- Stack size optimization

✅ **Error Handling**
- Consistent use of `esp_err_t`
- Error logging with ESP_LOG
- Graceful degradation (fallback messages)

✅ **Configuration Management**
- Two-layer config (build-time + runtime)
- NVS persistence
- CLI-based runtime updates

✅ **Documentation**
- Architecture documentation (ARCHITECTURE.md)
- Feature gap tracker (TODO.md)
- Code comments for complex logic

### 8.2 Weaknesses

⚠️ **Error Recovery**
- Limited retry logic
- No circuit breaker pattern
- No graceful degradation for API failures

⚠️ **Testing**
- No unit tests visible
- No integration tests
- Manual testing only

⚠️ **Code Duplication**
- Similar patterns in tool implementations
- Repeated JSON parsing logic
- No shared utilities

⚠️ **Hardcoded Values**
- Magic numbers (10 iterations, 20 messages, 3 days)
- Fixed buffer sizes
- No configuration for limits

⚠️ **Memory Safety**
- Manual memory management (potential leaks)
- No bounds checking in some string operations
- Stack overflow risk (fixed stack sizes)

---

## 9. Comparison with Reference (Nanobot)

MimiClaw is inspired by [Nanobot](https://github.com/HKUDS/nanobot) (Python implementation). Key differences:

| Feature | Nanobot | MimiClaw |
|---------|---------|----------|
| **Language** | Python | C |
| **Platform** | Linux/Server | ESP32-S3 (bare metal) |
| **Memory** | Unlimited | 8 MB PSRAM |
| **Storage** | Filesystem | SPIFFS (12 MB) |
| **Concurrency** | asyncio | FreeRTOS tasks |
| **LLM Provider** | Multi-provider (LiteLLM) | Anthropic only |
| **Tools** | Full set | 6 tools |
| **Media** | Full support | None |
| **Subagents** | Yes | No |
| **Cron** | Yes | No |
| **Skills** | Yes | No |

**MimiClaw's Achievement:** Implements core agent architecture in C on a microcontroller, achieving ~70% feature parity with Nanobot despite severe resource constraints.

---

## 10. Recommendations

### 10.1 High Priority (P0)

1. **Memory Write via Tool Use**
   - Expose `memory_write_long_term` and `memory_append_today` as tools
   - Add system prompt guidance on when to persist memory
   - Enable agent-driven memory persistence

2. **Telegram User Allowlist**
   - Store allow_from list in `mimi_secrets.h`
   - Filter in `process_updates()`
   - Critical for security (prevent API credit abuse)

3. **More Built-in Tools**
   - Add `message` tool (send to other chats)
   - Add `memory_write` tool (agent-driven persistence)
   - Complete file operations (append, dir creation)

### 10.2 Medium Priority (P1)

4. **Telegram Markdown to HTML**
   - Implement simplified converter
   - Or switch to `parse_mode: HTML`
   - Prevents send failures with special characters

5. **Bootstrap File Completion**
   - Add AGENTS.md (behavior guidelines)
   - Add TOOLS.md (tool documentation)
   - Align with Nanobot structure

6. **Longer Memory Lookback**
   - Make configurable (not fixed 3 days)
   - Add token budget awareness
   - Adaptive truncation

7. **Telegram Media Handling**
   - Image support (base64 for Claude Vision)
   - Voice transcription (Whisper API)
   - Document handling

### 10.3 Low Priority (P2)

8. **Cron / Heartbeat**
   - FreeRTOS timer for scheduled tasks
   - Periodic HEARTBEAT.md checks
   - Simple "every N minutes" support

9. **Multi-LLM Provider**
   - Abstract LLM interface
   - Support OpenAI-compatible API
   - Provider selection via config

10. **WebSocket Streaming**
    - Add `{"type":"token","content":"..."}` streaming
    - Real-time token push
    - Better UX for long responses

11. **Session Metadata**
    - Add created_at, updated_at to session files
    - Session expiration/cleanup
    - Conversation summarization

### 10.4 Code Quality

12. **Error Recovery**
    - Retry logic for transient API failures
    - Circuit breaker pattern
    - Graceful degradation

13. **Testing**
    - Unit tests for core modules
    - Integration tests for agent loop
    - Automated testing pipeline

14. **Configuration**
    - Make limits configurable (iterations, messages, days)
    - Runtime configuration for all limits
    - Token budget management

---

## 11. Conclusion

MimiClaw is an impressive achievement in embedded AI development. It successfully implements a complete AI agent architecture on a $5 microcontroller, providing:

- ✅ Full ReAct agent loop with tool use
- ✅ Multi-channel communication (Telegram, WebSocket, CLI)
- ✅ Local memory persistence
- ✅ Runtime configuration
- ✅ OTA updates
- ✅ HTTP proxy support

**Key Strengths:**
- Clean modular architecture
- Efficient resource usage (PSRAM allocation)
- Comprehensive documentation
- Runtime configuration flexibility

**Key Limitations:**
- Hardware constraints (memory, CPU, storage)
- Missing features (memory write via tools, media handling)
- Security gaps (no user allowlist)
- Limited error recovery

**Overall Assessment:**
MimiClaw demonstrates that sophisticated AI agent systems can run on resource-constrained embedded hardware. While it has limitations compared to server-based implementations, it achieves remarkable functionality within its constraints. The project is well-architected, documented, and provides a solid foundation for further development.

**Recommended Next Steps:**
1. Implement memory write via tool use (enables autonomous memory persistence)
2. Add Telegram user allowlist (critical security feature)
3. Complete bootstrap file alignment (AGENTS.md, TOOLS.md)
4. Add more built-in tools (message, memory_write)
5. Improve error recovery and retry logic

---

**Report Generated:** 2025-01-27  
**Analyzed Files:** 30+ source files, documentation, configuration  
**Analysis Depth:** Architecture, implementation, limitations, recommendations
