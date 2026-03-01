# starkbot-core-hyperpack

The core Hyperpack for [StarkBot](https://starkbot.ai) — an installable bundle of skills, agents, and modules that give a StarkBot instance its default capabilities.

## What is a Hyperpack?

A **Hyperpack** is a distributable package that extends a StarkBot agent with new capabilities. It bundles three component types into a single installable unit:

| Component | What it is | Format |
|-----------|-----------|--------|
| **Skills** | Instruction documents that teach the agent how to perform tasks | Markdown with YAML frontmatter |
| **Agents** | Persona definitions that give the AI specialized toolboxes and behaviors | Markdown with YAML frontmatter |
| **Modules** | Python microservices that run alongside the agent as background processes | Python (Flask) + TOML manifest |

A hyperpack is defined by a `hyperpack.toml` manifest at its root, which declares metadata and points to directories containing each component type.

## Hyperpack Manifest

```toml
[hyperpack]
name = "starkbot-core"
version = "0.1.0"
description = "Core skills, agents, and modules for StarkBot"

[contents]
skills = "skills"      # path to skills directory
agents = "agents"      # path to agents directory
modules = "modules"    # path to modules directory
```

### `[hyperpack]` fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique identifier for the hyperpack |
| `version` | string | Semver version |
| `description` | string | Human-readable description |

### `[contents]` fields

| Field | Type | Description |
|-------|------|-------------|
| `skills` | string | Relative path to skills directory |
| `agents` | string | Relative path to agents directory |
| `modules` | string | Relative path to modules directory |

---

## Skills

Skills are structured Markdown documents that teach the AI agent how to perform specific tasks. They are **not executable code** — they are instruction sets that the LLM follows at runtime.

### Directory structure

```
skills/
  swap/
    swap.md              # skill definition (Markdown + YAML frontmatter)
    abis/                # optional: Solidity ABI JSON files
    web3_presets.ron      # optional: blockchain call presets
    flows/               # optional: multi-step workflow documents
    references/          # optional: reference materials
```

### Skill frontmatter

```yaml
---
name: swap
description: "Swap tokens on Uniswap"
version: "1.0.0"
author: "starkbot"
tags: [crypto, defi, swap]
requires_tools: [web3_sendTransaction, use_skill]
arguments:
  token_in:
    description: "Token to swap from"
    required: true
  token_out:
    description: "Token to swap to"
    required: true
  amount:
    description: "Amount to swap"
    required: true
---

Instructions for the agent follow here in Markdown...
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique skill identifier (snake_case) |
| `description` | string | yes | What this skill does |
| `version` | string | yes | Semver version |
| `author` | string | yes | Skill author |
| `homepage` | string | no | URL for related service/docs |
| `metadata` | object | no | Arbitrary JSON metadata |
| `tags` | string[] | yes | Tags for skill discovery and agent routing |
| `requires_tools` | string[] | yes | Tool names the skill depends on |
| `arguments` | object | no | Named arguments with `description`, `required`, and `default` |

### How skills are used

1. An agent receives a user request
2. The agent matches the request to a skill by tags or name
3. The agent calls `use_skill` to load the skill's Markdown into context
4. The agent follows the instructions in the skill body, calling tools as needed

### Skills in this pack

This hyperpack includes 60 skills across several categories:

- **DeFi / Crypto**: `swap`, `transfer_eth`, `transfer_erc20`, `aave`, `uniswap`, `uniswap_lp`, `pendle`, `bridge_usdc`, `weth`, `token_price`, `dexscreener`, `geckoterminal`, `local_wallet`, `safe_wallet`, `bankr`, `nft_token`, `cryptopunks`, `ens`, `polymarket_us`
- **Identity / Payments**: `agent_identity`, `x402_payment`, `x402book`, `starkbot`
- **Social / Messaging**: `discord`, `telegram`, `twitter`, `moltworld`
- **Development**: `commit`, `code-review`, `create-project`, `deploy-github`, `github`, `github_discussions`, `claude_code`, `debug`, `test`, `plan`, `full-dev-workflow`, `create-skill`, `install_skill`
- **Infrastructure**: `railway`, `vercel`, `supabase`, `turso`, `cloudflare_dns`, `alchemy`, `firecrawl`, `figma`, `excalidraw`, `linear`, `remotion`
- **Agent Systems**: `heartbeat`, `impulse_map`, `impulse_evolver`, `notes`
- **Misc**: `weather`, `gog`, `starkhub`, `image_generation`

---

## Agents

Agents are persona definitions that give the AI different specializations. Each agent has access to specific tool groups and skill tags, controlling what it can do.

### Directory structure

```
agents/
  finance/
    agent.md             # agent definition (Markdown + YAML frontmatter)
    hooks/               # optional: event-driven hook templates
      heartbeat.md
      discord_message.md
```

### Agent frontmatter

```yaml
---
key: finance
version: "1.0.0"
label: "Finance"
emoji: "💰"
description: "Handles DeFi operations, token swaps, transfers, and portfolio management"
aliases: [defi, trading, crypto]
sort_order: 1
enabled: true
max_iterations: 90
skip_task_planner: false
hidden: false
tool_groups: [system, web, filesystem, finance]
skill_tags: [crypto, defi, swap, transfer, wallet, nft]
additional_tools: []
---

Agent persona instructions follow here...
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key` | string | yes | Unique agent identifier |
| `version` | string | yes | Semver version |
| `label` | string | yes | Display name |
| `emoji` | string | yes | Icon emoji |
| `description` | string | yes | What this agent does |
| `aliases` | string[] | no | Alternative names for routing |
| `sort_order` | integer | yes | Display ordering (-1 = first, 999 = hidden) |
| `enabled` | boolean | yes | Whether agent is active |
| `max_iterations` | integer | yes | Max LLM iteration loops |
| `skip_task_planner` | boolean | yes | Handle tasks directly without planner |
| `hidden` | boolean | yes | Hide from UI (for system agents) |
| `tool_groups` | string[] | yes | Tool categories available to this agent |
| `skill_tags` | string[] | yes | Which skill tags this agent can access |
| `additional_tools` | string[] | no | Extra individual tools beyond groups |

### How agents are routed

The **Director** agent orchestrates all others. When a request comes in:

1. Director examines the request and selects the appropriate specialist agent
2. For single-domain tasks, Director switches to that agent
3. For multi-domain tasks, Director can spawn parallel sub-agents
4. Each agent only sees skills matching its `skill_tags`

### Hooks

Hooks are event-driven templates that activate agents on specific events. Hook files are Markdown templates with placeholder variables (`{data}`, `{timestamp}`, `{guildId}`, etc.) that get injected at runtime.

Hook types:
- `heartbeat.md` — Periodic timer-based triggers
- `discord_message.md` — Triggered on new Discord messages
- `telegram_message.md` — Triggered on new Telegram messages
- `*_pulse.md` — Module-specific periodic pulses
- `*_sign_tx.md` — Transaction signing hooks

### Agents in this pack

| Agent | Key | Description | Hidden |
|-------|-----|-------------|--------|
| Director | `director` | Orchestrator — routes tasks to specialists | No |
| Finance | `finance` | DeFi operations, swaps, transfers | No |
| Code Engineer | `code_engineer` | Code editing, git, testing, deployment | No |
| Secretary | `secretary` | Social media, messaging, scheduling, image gen | No |
| Discord Moderator | `discord_moderator` | Autonomous spam/scam detection with 3-strike bans | Yes |
| Telegram Moderator | `telegram_moderator` | Autonomous spam/scam detection with 3-strike bans | Yes |
| Impulse Evolver | `impulse_evolver` | Autonomous impulse map evolution | Yes |

---

## Modules

Modules are standalone Python microservices (Flask apps) that run alongside the StarkBot backend as separate processes. They add domain-specific capabilities with their own databases, dashboards, and RPC APIs.

### Directory structure

```
modules/
  wallet_monitor/
    module.toml          # module manifest (required)
    service.py           # Flask application (required)
    dashboard.py         # dashboard UI class (optional)
    skill.md             # or skill/ directory — teaches the agent to use this module
    agent/               # optional: embedded autonomous agent + hooks
      agent.md
      hooks/
        heartbeat.md
```

### Module manifest (`module.toml`)

```toml
[module]
name = "wallet_monitor"
version = "1.1.0"
author = "starkbot"
description = "Monitor ETH wallets on Mainnet and Base"

[service]
command = "uv run service.py"
default_port = 9100
port_env_var = "WALLET_MONITOR_PORT"
has_dashboard = true
dashboard_styles = ["tui", "html"]
health_endpoint = "/rpc/status"
dashboard_endpoint = "/"
backup_endpoint = "/rpc/backup/export"
restore_endpoint = "/rpc/backup/restore"

[service.env_vars]
ALCHEMY_API_KEY = { required = true, description = "Alchemy API key" }

[skill]
content_file = "skill.md"
# OR: skill_dir = "skill"

[agent]
dir = "agent"
```

### `[module]` fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique module identifier |
| `version` | string | Semver version |
| `author` | string | Module author |
| `description` | string | Human-readable description |

### `[service]` fields

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | Shell command to start the service |
| `default_port` | integer | Default listening port |
| `port_env_var` | string | Env var to override port |
| `has_dashboard` | boolean | Whether it has a web dashboard |
| `dashboard_styles` | string[] | Dashboard rendering modes (`tui`, `html`) |
| `health_endpoint` | string | Health check endpoint path |
| `dashboard_endpoint` | string | Dashboard URL path |
| `backup_endpoint` | string | Data export endpoint |
| `restore_endpoint` | string | Data import endpoint |

### `[service.env_vars]` fields

Key-value pairs where each key is an environment variable name and the value is an object with:
- `required` (boolean) — whether the variable must be set
- `description` (string) — human-readable description

### `[skill]` fields

| Field | Type | Description |
|-------|------|-------------|
| `content_file` | string | Path to a single skill Markdown file |
| `skill_dir` | string | Path to a directory of skill files (use one or the other) |

### `[agent]` fields

| Field | Type | Description |
|-------|------|-------------|
| `dir` | string | Path to directory containing `agent.md` and `hooks/` |

### Module communication pattern

All modules expose JSON-RPC endpoints under `/rpc/...`. The LLM interacts with modules via the `local_rpc` tool, which routes requests to `http://127.0.0.1:{port}/rpc/...`. Standard endpoints include:

- `/rpc/status` — health check
- `/rpc/backup/export` — full data export
- `/rpc/backup/restore` — data import
- Custom domain endpoints (e.g., `/rpc/decision`, `/rpc/portfolio`)

### Module lifecycle

1. StarkBot discovers `module.toml` files at startup
2. Assigns ports and injects environment variables
3. Spawns each service as a subprocess (`uv run service.py`)
4. Health-checks each service via `health_endpoint`
5. Module background workers fire hooks to wake the agent
6. Agent calls back into module RPC endpoints to take action

### Modules in this pack

| Module | Port | Description |
|--------|------|-------------|
| `wallet_monitor` | 9100 | Monitor ETH wallets on Mainnet + Base |
| `discord_tipping` | 9101 | Discord user profiles and wallet linking |
| `kv_store` | 9103 | Persistent key/value store for agent state |
| `spot_trader` | 9104 | Autonomous DeFi spot trader on Base |
| `perps_trader` | 9105 | Autonomous perpetual futures trader (Avantis) |
| `whale_tracker` | 9106 | Whale wallet tracker with scored alerts |
| `twitter_watcher` | 9108 | Watch Twitter accounts for new tweets |
| `meta_marketer` | 9110 | Meta (Facebook/Instagram) ads manager |
| `hyper_claw` | 9111 | Perpetual futures trader (Orderly Network) |
| `starkbot_sdk` | — | Shared Python SDK for all modules |

---

## Installation

Point your StarkBot instance at a hyperpack Git repository URL. The backend clones the repo, reads `hyperpack.toml`, and loads all skills, agents, and modules from the declared directories.

```
~/.starkbot/
  hyperpacks/
    starkbot-core/          # cloned from this repo
      hyperpack.toml
      skills/
      agents/
      modules/
```

## License

See [LICENSE](LICENSE) for details.
