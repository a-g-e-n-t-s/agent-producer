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
  - [broker.remote]
    - URL = "wss://broker.dadavidtseng.com/kadi" — remote broker URL used by the agent (local broker is optional and may be configured as [broker.local] with URL and NETWORKS)
    - NETWORKS = ["chatbot", "quest", "producer", "file", "global", "voice-services"] — networks to join on the remote broker
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
- agent.json deploy.secrets defines vaults and required keys; when deployed to Akash the agent expects secrets delivered via the broker (see agent.json deploy stanza).

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
- agent.json — agent metadata, scripts, build & deploy settings (see updated build/deploy for secret vaults and Akash command)
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
  - MemoryService: persistent memory stored in DATA_PATH (./data/memory)
  - Graceful shutdown handling for SIGINT/SIGTERM

- Tool registration
  - registerAllTools (imported from ./tools/index.js) registers each tool (echo, list_tools, quest_approve, etc.) with KadiClient via kadiClient.registerTool()

- Upstream integration
  - The agent forwards task management to an MCP upstream (mcp-shrimp-task-manager) using kadiClient.load() so existing task managers can handle planning/splitting.

- Event publishing and orchestration
  - After forwarding or creating tasks, agent-producer publishes events (e.g., `quest.tasks_ready`) to the broker.
  - agent-lead listens to these events and assigns tasks to worker agents by role.
  - Workers execute tasks and commit results; agent-lead verifies and publishes verification events (e.g., task.verified).
  - agent-producer relays status updates back to human channels (Slack, Discord) and to the originator (Claude Code/Desktop).

- Multi-broker support
  - Primary broker is selected from [broker.local] (preferred for dev) or [broker.remote] via config.toml or the KADI_BROKER_URL_LOCAL / KADI_BROKER_URL_REMOTE env vars.
  - If both local and remote brokers are configured, the non-primary broker is registered as an additional broker.

- Tool lifecycle
  - Some tools are immediate/one-shot (e.g., list_tools). Others are stateful and interact with quest/task lifecycles managed by upstream task manager components.

Deployment notes:
- agent.json includes a build stanza (base image node:20-alpine) and an Akash deploy configuration that exposes port 3000, sets NODE_ENV=production, and declares required secrets and resource requests.
- The build now runs kadi install kadi-secret and kadi install as part of the container build flow.
- The Akash service command fetches secrets from configured vaults before starting:
  "kadi secret receive --vault anthropic --vault model-manager --vault arcadedb && kadi run start"
- When deploying to KADI/Akash, ensure vaults provide required secrets (ANTHROPIC_API_KEY, MODEL_MANAGER_API_KEY, MODEL_MANAGER_BASE_URL, ARCADE_USERNAME, ARCADE_PASSWORD) and delivery is configured via the broker.

Development
-----------
Useful npm scripts (defined in agent.json):
- `npm run setup` — transpile TypeScript (npx tsc)
- `npm start` — run compiled build: node dist/index.js
- `npm run dev` — run in watch mode (npx tsx watch src/index.ts)
- `npm run build` — compile TypeScript (npx tsc)
- `npm run type-check` — run tsc without emitting
- `npm run lint` — run ESLint on src
- `npm run test` — run tests (vitest)
- `npm run clean` — remove node_modules, dist, and lock files

Build container image (as configured in agent.json.build.default):
- Steps performed by the build:
  - npm ci --include=dev
  - kadi install kadi-secret
  - kadi install
  - npx tsc
  - npm prune --omit=dev
- Base image: node:20-alpine

Local debugging tips:
- Use `npm run dev` to get hot-reload behavior via tsx.
- Provide environment variables locally via .env or by exporting before starting:
  - export KADI_BROKER_URL_REMOTE="wss://broker.dadavidtseng.com/kadi"
  - export ANTHROPIC_API_KEY="sk-..."
  - export MODEL_MANAGER_API_KEY="..."
- To inspect memory, view files under ./data/memory.

References in source:
- Main bootstrap: src/index.ts (registers tools, loads config, loads vault credentials)
- Tool registration entrypoint: ./tools/index.js
- Tool schemas referenced in src/index.ts: plan-task.js, list-tasks.js, task-status.js, assign-task.js

If you need example tool implementations, a deployment manifest for Akash, or a walkthrough for enabling LLM features (Model Manager primary + Anthropic fallback), tell me which part you want expanded and I will add targeted instructions.

## Quick Start

```bash
cd agent-producer
npm install
kadi install
kadi run start
```

## Tools

<!-- Tools listed above in the Tools section -->

## Configuration

### agent.json

See agent.json in the repository for the canonical build and deploy settings (version 0.3.25 as of this source). Notable fields:
- entrypoint: dist/index.js
- build.default.from: node:20-alpine
- build.run: includes kadi install kadi-secret and kadi install
- deploy.akash.services.app.command: receives secrets from vaults (anthropic, model-manager, arcadedb) before starting
- deploy.akash.secrets.vaults: anthropic, model-manager, arcadedb (required keys listed in agent.json)

### Abilities

- `secret-ability` *
- `ability-log` ^0.1.5

### Brokers

- **remote**: `wss://broker.dadavidtseng.com/kadi`

## Architecture

<!-- Architecture content summarized above -->

## Development

```bash
npm install
npm run build
kadi run start
```

---