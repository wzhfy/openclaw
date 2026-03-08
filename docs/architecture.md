# Technical Architecture

This document describes the overall technical architecture of OpenClaw using Mermaid diagrams, covering all layers and their interactions.

## System Overview

```mermaid
graph TB
    subgraph Clients["Client Layer"]
        direction LR
        IOS["iOS App\n(Swift/SwiftUI)"]
        AND["Android App\n(Kotlin/Jetpack)"]
        MAC["macOS App\n(Swift/Menu Bar)"]
        WUI["Web UI\n(React/TypeScript)"]
        CLI["CLI\n(openclaw)"]
    end

    subgraph Gateway["Gateway Layer (src/gateway/)"]
        direction TB
        HTTP["HTTP Server\n:18789"]
        WS["WebSocket Server\nreal-time push"]
        AUTH["Auth\nauthentication"]
        CRON["Cron\nscheduled jobs"]
        RPC["RPC Methods\n100+ methods"]
        PLG_LOAD["Plugin Loader"]
    end

    subgraph Channels["Channel Layer"]
        direction LR
        subgraph CoreCh["Built-in channels"]
            TG["Telegram"]
            WA["WhatsApp"]
        end
        subgraph ExtCh["Extension channels (extensions/)"]
            DC["Discord"]
            SL["Slack"]
            SIG["Signal"]
            IM["iMessage"]
            ZALO["Zalo"]
            MAT["Matrix"]
            MST["MS Teams"]
            MORE["...41 channels total"]
        end
    end

    subgraph Routing["Routing Layer (src/routing/)"]
        RESOLVE["Route resolution\nresolve-route.ts"]
        SESSION_KEY["Session key generation\nsession-key.ts"]
        ACCT["Account lookup\naccount-lookup.ts"]
    end

    subgraph Agents["Agent Layer (src/agents/)"]
        direction TB
        ACP["ACP subprocess\nacp-spawn.ts"]
        SCOPE["Agent config\nagent-scope.ts"]
        TOOLS["Tool system\ntools/"]
        SKILLS["Skill system\nskills/"]
        CTX["Context engine\ncontext-engine/"]
        MEDIA["Media understanding\nmedia-understanding/"]
    end

    subgraph Providers["AI Provider Layer (src/providers/)"]
        direction LR
        OAI["OpenAI\nGPT-4o"]
        ANT["Anthropic\nClaude 3"]
        GEM["Google\nGemini"]
        MIS["Mistral"]
        OLL["Ollama\n(local)"]
        EXT_P["Others\nQwen/MiniMax/..."]
    end

    subgraph Memory["Memory / RAG Layer (src/memory/)"]
        direction TB
        MGR["Memory Manager"]
        SQLITE["SQLite + sqlite-vec\nvector store"]
        LANCE["LanceDB\n(optional)"]
        EMBED["Embeddings"]
        SEARCH["Hybrid search\nsemantic + keyword"]
    end

    subgraph Storage["Persistence Layer"]
        direction LR
        CONFIG["Config file\n~/.config/openclaw/config.yaml"]
        SESSIONS["Session history\n~/.openclaw/sessions/"]
        CREDS["Credentials\n~/.openclaw/credentials/"]
        SECRETS["Secrets vault\nsrc/secrets/"]
    end

    subgraph Plugins["Plugin / Extension System (extensions/)"]
        direction LR
        SDK["Plugin SDK\nsrc/plugin-sdk/"]
        MANIFEST["openclaw.plugin.json\nplugin manifest"]
        CH_ADAPTER["Channel adapters\n10+ adapter interfaces"]
    end

    %% Clients -> Gateway
    IOS  -->|"WebSocket"| WS
    AND  -->|"WebSocket"| WS
    MAC  -->|"WebSocket"| WS
    WUI  -->|"WebSocket"| WS
    CLI  -->|"HTTP/WS"| HTTP

    %% Channels -> Gateway (inbound)
    CoreCh -->|"Webhook/Poll\nHTTP POST"| HTTP
    ExtCh  -->|"Webhook/Poll\nHTTP POST"| HTTP

    %% Gateway internals
    HTTP --> AUTH
    HTTP --> RPC
    WS   --> RPC
    RPC  --> PLG_LOAD
    RPC  --> CRON

    %% Gateway -> Routing
    RPC --> RESOLVE
    RESOLVE --> SESSION_KEY
    RESOLVE --> ACCT

    %% Routing -> Agent
    RESOLVE --> ACP
    ACP     --> SCOPE
    ACP     --> TOOLS
    ACP     --> SKILLS
    ACP     --> CTX
    CTX     --> MEDIA

    %% Agent -> AI providers
    ACP --> OAI
    ACP --> ANT
    ACP --> GEM
    ACP --> MIS
    ACP --> OLL
    ACP --> EXT_P

    %% Agent -> Memory
    ACP    --> MGR
    MGR    --> SQLITE
    MGR    --> LANCE
    MGR    --> EMBED
    MGR    --> SEARCH

    %% Gateway -> outbound channels
    RPC -->|"outbound send"| CoreCh
    RPC -->|"outbound send"| ExtCh

    %% Plugin system
    SDK       --> CH_ADAPTER
    MANIFEST  --> PLG_LOAD
    CH_ADAPTER --> CoreCh
    CH_ADAPTER --> ExtCh

    %% Persistence
    RPC    --> CONFIG
    RPC    --> SESSIONS
    AUTH   --> CREDS
    AUTH   --> SECRETS
```

---

## Message Processing Flow

```mermaid
sequenceDiagram
    participant U as User<br/>(WhatsApp/Telegram/Discord...)
    participant CH as Channel Extension
    participant GW as Gateway HTTP
    participant RT as Routing Engine
    participant AG as Agent subprocess<br/>(ACP)
    participant LLM as AI Provider<br/>(OpenAI/Claude/Gemini)
    participant MEM as Memory / RAG
    participant WS as WebSocket<br/>(Native Apps/Web UI)

    U->>CH: Send message
    CH->>GW: POST /channel/{name}/inbound<br/>ChannelInboundMessage
    GW->>GW: Validate API key / auth
    GW->>RT: Resolve route
    RT->>RT: Determine agentId + sessionKey<br/>(by peer/guild/role/default)
    RT->>AG: Start/resume agent subprocess

    AG->>MEM: Retrieve relevant memories<br/>(vector semantic search)
    MEM-->>AG: Return context memories

    AG->>AG: Assemble context<br/>(history + memories + media)
    AG->>LLM: Call model API<br/>(streaming/batch)
    LLM-->>AG: Stream tokens back
    AG-->>WS: Push streaming text in real-time
    AG-->>AG: Execute tool calls

    AG->>GW: Return final response + tool results
    GW->>GW: Format output (Markdown/media)
    GW->>CH: Outbound send
    CH->>U: Deliver reply

    GW->>MEM: Store new memory
    GW->>GW: Persist session history<br/>(sessions/*.json)
    GW->>WS: Broadcast status update
```

---

## Plugin Architecture

```mermaid
graph LR
    subgraph Core["Core (src/)"]
        SDK["Plugin SDK\n(src/plugin-sdk/)"]
        REG["Channel Registry\n(src/channels/registry.ts)"]
        DOCK["Channel Dock\nlifecycle state machine"]
    end

    subgraph PluginManifest["Plugin Manifest"]
        PM["openclaw.plugin.json\nid / channels / configSchema"]
    end

    subgraph Adapters["Channel Adapter Interfaces (10+)"]
        MA["messagingAdapter\nsend/receive messages"]
        OA["outboundAdapter\noutbound delivery"]
        AA["authAdapter\nlogin / auth"]
        SA["statusAdapter\nhealth checks"]
        SEC["securityAdapter\nACL / allowlists"]
        PA["pairingAdapter\ndevice pairing"]
        GA["groupAdapter\ngroup management"]
        RA["resolverAdapter\nuser resolution"]
        CMD["commandAdapter\nnative commands"]
    end

    subgraph ExtExamples["Extension Examples"]
        E1["extensions/telegram/"]
        E2["extensions/discord/"]
        E3["extensions/slack/"]
        E4["extensions/matrix/"]
        E5["extensions/memory-lancedb/"]
        E6["extensions/voice-call/"]
    end

    SDK --> Adapters
    PM --> REG
    REG --> DOCK
    Adapters --> E1
    Adapters --> E2
    Adapters --> E3
    Adapters --> E4
    E5 -->|"memory backend"| SDK
    E6 -->|"voice pipeline"| SDK
```

---

## Configuration and Storage Architecture

```mermaid
graph TB
    subgraph ConfigSystem["Config System (src/config/)"]
        YAML["config.yaml\n(JSON5, YAML)"]
        ZOD["Zod schema validation\nschema.ts"]
        IO["config/io.ts\nload + env var substitution"]
        PATHS["paths.ts\ndirectory path resolution"]
        MIGRATE["v1 to v2 migration\nlegacy migration"]
    end

    subgraph SessionStore["Session Store"]
        SESS_FILE["~/.openclaw/sessions/\n{sessionKey}.json"]
        SESS_MGR["Session Manager\nincremental append"]
    end

    subgraph MemoryStore["Vector Memory Store"]
        SQLITE_VEC["SQLite + sqlite-vec\nprimary store"]
        LANCE_DB["LanceDB\n(optional backend)"]
        EMBED_API["Embedding API\nOpenAI / Gemini / Voyage / Mistral"]
    end

    subgraph SecretsStore["Credential Store"]
        CRED_FILE["~/.openclaw/credentials/\n{profileId}.json"]
        ENV_VAR["Environment variables\nOPENAI_API_KEY etc."]
        OAUTH["OAuth token\nauto-refresh"]
        KEYCHAIN["System keychain\n(macOS/Windows)"]
    end

    YAML --> IO
    IO --> ZOD
    IO --> PATHS
    MIGRATE --> YAML
    IO --> SESS_MGR
    SESS_MGR --> SESS_FILE
    IO --> CRED_FILE
    CRED_FILE --> ENV_VAR
    CRED_FILE --> OAUTH
    CRED_FILE --> KEYCHAIN
    IO -.->|"vector search"| SQLITE_VEC
    SQLITE_VEC -.->|"optional swap"| LANCE_DB
    EMBED_API --> SQLITE_VEC
```

---

## Multi-Platform Client Architecture

```mermaid
graph LR
    subgraph GW_SERVER["Gateway Service :18789"]
        GWS["WebSocket Hub\n+ HTTP REST"]
    end

    subgraph NativeApps["Native Apps"]
        IOS_APP["iOS App\n(Swift/SwiftUI)\napps/ios/"]
        AND_APP["Android App\n(Kotlin/Jetpack)\napps/android/"]
        MAC_APP["macOS App\n(Swift menu bar)\napps/macos/"]
    end

    subgraph WebLayer["Web Console"]
        REACT["React Web UI\nui/"]
        WEB_WS["WebSocket Client"]
    end

    subgraph CLILayer["Command Line"]
        CLI_CMD["openclaw CLI\nsrc/cli/"]
        CLI_WS["HTTP/WS Client"]
    end

    GWS -->|"real-time push\nmessages/status/streaming"| IOS_APP
    GWS -->|"real-time push"| AND_APP
    GWS -->|"real-time push"| MAC_APP
    IOS_APP -->|"WebSocket"| GWS
    AND_APP -->|"WebSocket"| GWS
    MAC_APP -->|"WebSocket"| GWS

    REACT --> WEB_WS
    WEB_WS -->|"WebSocket"| GWS
    GWS -->|"serve /\nstatic files"| REACT

    CLI_CMD --> CLI_WS
    CLI_WS -->|"HTTP + WS streaming"| GWS
```

---

## Component Responsibilities

| Layer | Module | Responsibility |
|-------|--------|----------------|
| **Client** | iOS / Android / macOS App | Native UI, persistent WebSocket connection, offline message queue |
| **Client** | Web UI (React) | Browser-based control plane, real-time config, model management |
| **Client** | CLI | Command-line interface, scripting, development tooling |
| **Gateway** | HTTP Server | REST endpoints, WebSocket upgrade, plugin HTTP routing |
| **Gateway** | WebSocket Server | Bidirectional real-time push, client state sync |
| **Gateway** | RPC Methods | 100+ methods: chat / config / memory / channels / plugins |
| **Gateway** | Cron Service | Scheduled tasks, auto-replies, periodic maintenance |
| **Channel** | Channel Extensions | Message adapter, inbound webhook/polling, outbound formatting |
| **Routing** | Routing Engine | Message-to-agent mapping, session key generation, multi-agent dispatch |
| **Agent** | ACP subprocess | LLM calls, tool execution, streaming responses, session management |
| **Agent** | Tools / Skills | Pluggable tools (search, code execution, file I/O, etc.) |
| **AI Provider** | Provider integrations | OpenAI / Anthropic / Gemini / Mistral / Ollama / ... |
| **Memory** | Memory / RAG | Vector search, semantic retrieval, long-term memory persistence |
| **Persistence** | Config / Sessions / Secrets | YAML config, JSON session history, encrypted credentials |
| **Plugin System** | Plugin SDK + Manifest | Standardized extension interface, dynamic loading, HTTP route mounting |
