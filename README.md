# agent-producer
> KĀDI "producer" agent: orchestrates quests and tasks, registers tools with KĀDI broker, and forwards task lifecycle events to worker agents and human channels.

Overview
--------
agent-producer is a KĀDI agent (type: agent) that coordinates multi-agent work (artist, designer, programmer) via the KĀDI event-driven protocol. It registers tools with the broker, forwards task management to upstream task managers, publishes task assignment events (e.g., quest.tasks_ready), and relays status updates to human channels (Slack/Discord). The project uses agents-library for common agent primitives and can enable LLM-backed features when ANTHROPIC_API_KEY and model manager credentials are provided.

Quick Start
-----------
1. Clone the repository and install Node dependencies:
`npm install`

2. Install the agent into your local KĀDI workspace (uses KĀDI CLI):
`kadi install`

3. Start the agent via the KĀDI runner:
`kadi run start`

Alternative local development:
- Build TypeScript and run:
  `npm run setup`
  `npm start`

- Run in watch/dev mode:
  `npm run dev`

Notes:
- The agent reads configuration from config.toml (walk-up discovery). Environment variables (.env) take precedence for overrides.
- Secrets should be stored in your vault/secrets.toml or injected by your deployment (see Configuration section).
- Container/build behavior: the image build installs the kadi-secret helper during the build (see agent.json build.run). When deployed to Akash the deployed command invokes kadi secret receive to fetch required vault credentials before starting the agent:
  `kadi secret receive --vault anthropic --vault model-manager --vault arcadedb && kadi run start`
  This is how the required secrets (see Deploy section) are delivered to the running instance.

Tools
-----
Registered tools (as exported by ./tools/index.js):
- echo: Echo back the input text with its length (placeholder tool - replace with your own)
- my_tool: What this tool does
- echo: ... (duplicate placeholder registration present in source; keep or replace as needed)
- list_tools: List all available tools in human-readable format. This is an informational query — it does NOT complete the user.
- quest_approve: Approve a quest plan. The quest must be in pending_approval status. After approval, the quest moves to approved status and is ready for task splitting.
- quest_request_revision: Request revision of a quest plan. The quest must be in pending_approval status. Feedback is required to explain what needs to change. The quest returns to draft status for revision.
- quest_reject: Reject a quest plan. The quest must be in pending_approval status. Feedback is required to explain the rejection reason. The quest moves to rejected status.
- task_approve: Approve a completed task. The task must be in pending_approval status. After approval, the task moves to completed status.
- task_request_revision: Request revision of a task result. The task must be in pending_approval status. Feedback is required. The task returns to in_progress status for the agent to revise.
- task_reject: Reject a task result. The task must be in pending_approval status. Feedback is required. The task moves to failed status.
- task_execution: Trigger task execution by publishing quest.tasks_ready for agent-lead to assign tasks to worker agents.

Configuration
-------------
Primary configuration files and fields (match config.toml keys; non-secret settings committed to git):

- config.toml (committed non-secret configuration)
  - [agent]
    - ID = "agent-producer"
    - ROLE = "producer"
    - VERSION = "0.3.4"
  - [logging]
    - LEVEL = "debug"
  - [broker.local]
    - URL = "wss://broker.dadavidtseng.com/kadi"
    - NETWORKS = ["chatbot", "quest", "producer", "file", "global", "voice-services"]
  - [broker.remote]
    - URL = "wss://broker.kadi.build/kadi"
    - NETWORKS = ["chatbot", "quest", "producer", "file", "global", "voice-services"]
  - [bot]
    - TOOL_TIMEOUT_MS = 10000 — default per-tool timeout in milliseconds
  - [bot.slack]
    - ENABLED = true
    - USER_ID = "U09SCDV78AK"
  - [bot.discord]
    - ENABLED = true
    - USER_ID = "1438685741751210025"
  - [memory]
    - DATA_PATH = "./data/memory" — local path for persistent memory
  - [secrets]
    - VAULTS = ["anthropic", "model-manager", "arcadedb"]
    - KEYS = ["MODEL_MANAGER_BASE_URL", "MODEL_MANAGER_API_KEY", "ANTHROPIC_API_KEY", "ARCADE_USERNAME", "ARCADE_PASSWORD"]
  - [arcadedb]
    - HOST = "arcadedb.dadavidtseng.com"
    - PORT = 443
    - USERNAME = "root"
    - DATABASE = "agents_logs"
  - [provider]
    - PRIMARY = "model-manager"
    - FALLBACK = "anthropic"
  - [provider.model-manager]
    - MODEL = "gpt-5-mini"
  - [provider.anthropic]
    - MODEL = "claude-haiku-4-5-20251001"

Secret management:
- Secrets should be stored in your vault or secrets.toml (encrypted) and NOT committed to git.
- .env may be used for local overrides (not committed).
- agent.json deploy.secrets defines vaults and required keys; when deployed to Akash the agent expects secrets delivered via the broker (see agent.json deploy stanza). The Akash deploy command in agent.json explicitly runs:
  `kadi secret receive --vault anthropic --vault model-manager --vault arcadedb && kadi run start`
  Required vault keys (per agent.json):
  - anthropic: ANTHROPIC_API_KEY
  - model-manager: MODEL_MANAGER_API_KEY, MODEL_MANAGER_BASE_URL
  - arcadedb: ARCADE_USERNAME, ARCADE_PASSWORD

Environment variables (overrides):
- KADI_BROKER_URL_LOCAL — override local broker URL (used when [broker.local] is present)
- KADI_BROKER_URL_REMOTE — override remote broker URL (used when [broker.remote] is present)
- KADI_NETWORK_LOCAL — comma-separated networks override for local broker (e.g., chatbot,producer,quest)
- KADI_NETWORK_REMOTE — comma-separated networks override for remote broker
- ANTHROPIC_API_KEY — enables Anthropic LLM-backed features when present (can be loaded from vault as well)
- MODEL_MANAGER_BASE_URL — optional model manager base URL (also expected in model-manager vault)
- MODEL_MANAGER_API_KEY — model manager API key
- ARCADE_HOST / ARCADE_PORT — optional ArcadeDB host/port (used in deployment environment)
- (Vault-provided) ARCADE_USERNAME, ARCADE_PASSWORD — ArcadeDB credentials (kept secret)

Relevant files / paths:
- agent.json — agent metadata, scripts, build & deploy settings. Note: build installs kadi-secret during image build and the Akash deploy command fetches vault secrets via `kadi secret receive --vault anthropic --vault model-manager --vault arcadedb` before starting the agent. The agent.json package version and Akash image tag are 0.3.25 in the source.
- config.toml — agent configuration (non-secret)
- secrets.toml — (encrypted) secrets file; use vault for production
- .env — optional local overrides
- src/index.ts — main agent bootstrap
- src/tools/* and ./tools/index.js — tool registrations and orchestrator injectors
- ./data/memory — persistent memory directory used by agents-library MemoryService

Architecture
------------
High-level components and data flow:
- BaseAgent (agents-library)
  - KadiClient: connects to one or more KĀDI brokers and registers tools
  - ProviderManager: optional LLM provider (model-manager primary / anthropic fallback) when credentials are configured
  - MemoryService: persistent memory stored in