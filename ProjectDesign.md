# OpenClaw — Project Design & Implementation Plan

**Platform:** Arch Linux (personal PC)
**Language:** Python 3.11+ / Bash (setup)
**Interface:** Private Discord channel (sole command surface)
**Status:** Planning Phase

---

## 1. Guiding Principles

This is a single-user personal tool. Every design decision flows from that:

- **No inbound ports.** Discord's outbound websocket is the only network surface. No tunnels, no exposed services.
- **No input sanitization.** You are the only user; every message is trusted.
- **No hardware monitoring layer.** The PC is your daily driver; you already know when it's unhappy.
- **Agents are disposable specialists.** Each task type gets its own agent with its own system prompt and conversation history, sidestepping catastrophic forgetting entirely.
- **LAN LLM first, cloud fallback.** The llama.cpp-server instance is intermittent, so the gateway must health-check it and fall through to Claude or OpenAI transparently.

---

## 2. Architecture Overview

```
You (phone/laptop/anywhere)
        │
        ▼
┌─────────────────────────────┐
│  Private Discord Channel    │
│  (commands in, responses out)│
└──────────────┬──────────────┘
               │ outbound websocket (discord.py)
               ▼
┌──────────────────────────────────────────────────┐
│                  Discord Bot                     │
│  • Listens on one channel                        │
│  • Parses command prefix or slash commands        │
│  • Streams long responses in chunks              │
│  • Sends file attachments for script output       │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│                 Orchestrator                      │
│  ┌──────────────┐  ┌───────────────────────┐     │
│  │ Command      │  │ Conversation          │     │
│  │ Parser       │  │ State (per-agent)     │     │
│  └──────┬───────┘  └───────────┬───────────┘     │
│         │                      │                  │
│  ┌──────▼──────────────────────▼───────────┐     │
│  │           Agent Router                  │     │
│  │  • /agent <name> <msg>  → route to agent│     │
│  │  • /newagent <name> <desc> → factory    │     │
│  │  • /agents              → list agents   │     │
│  │  • /kill <name>         → remove agent  │     │
│  │  • /run <file>          → execute script│     │
│  │  • /approve <id>        → confirm exec  │     │
│  │  • (bare message)       → default agent │     │
│  └──────┬──────────────────────────────────┘     │
└─────────┼────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────┐
│              Agent Registry                      │
│                                                  │
│  agents/                                         │
│  ├── _defaults.yaml      (shared constraints)    │
│  ├── chat.yaml           (general conversation)  │
│  ├── claw.yaml           (claw-machine control)  │
│  └── <user-created>.yaml (via /newagent)         │
│                                                  │
│  Each agent has:                                 │
│  • system_prompt          (personality + task)   │
│  • conversation_history[] (in-memory, ephemeral) │
│  • permissions{}          (read_fs, exec, net)   │
│  • model_preference       (optional override)    │
└──────────┬───────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────┐
│          LLM Gateway + Health Monitor            │
│                                                  │
│  Priority chain:                                 │
│  1. LAN llama.cpp-server  (http://localhost:8080)│
│  2. Claude API            (anthropic SDK)        │
│  3. OpenAI API            (openai SDK)           │
│                                                  │
│  • Pings LAN endpoint every 30s                  │
│  • Caches availability status                    │
│  • Exposes a single  complete(messages, model?)  │
│    that the rest of the codebase calls           │
│  • Handles streaming where supported             │
└──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────┐
│           Filesystem + Script Runner             │
│                                                  │
│  Read access:                                    │
│  • Agent can request file contents via the LLM   │
│    gateway (injected into context by orchestrator)│
│                                                  │
│  Script creation:                                │
│  • Agent outputs a fenced code block             │
│  • Orchestrator saves to scripts/<name>.py|.sh   │
│  • Execution requires your explicit /approve     │
│  • stdout/stderr captured and sent back to Discord│
└──────────────────────────────────────────────────┘
```

---

## 3. Directory Structure

```
pi-openclaw/
├── README.md
├── ProjectDesign.md
├── LICENSE
├── pyproject.toml              # uv/pip project metadata
├── .env.example                # template for secrets
├── .env                        # actual secrets (gitignored)
│
├── config/
│   ├── settings.yaml           # gateway URLs, model priority, paths
│   └── agents/
│       ├── _defaults.yaml      # shared agent constraints
│       ├── chat.yaml           # built-in: general assistant
│       └── claw.yaml           # built-in: claw-machine agent
│
├── src/
│   ├── __init__.py
│   │
│   ├── bot/
│   │   ├── __init__.py
│   │   ├── client.py           # discord.py bot setup, event loop
│   │   ├── commands.py         # command parsing and dispatch
│   │   └── formatting.py       # chunk long messages, attach files
│   │
│   ├── orchestrator/
│   │   ├── __init__.py
│   │   ├── router.py           # maps commands → agents
│   │   ├── state.py            # per-agent conversation history
│   │   └── permissions.py      # check what an agent is allowed to do
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── registry.py         # load/list/create/delete agents
│   │   ├── factory.py          # /newagent: generate agent YAML
│   │   └── base.py             # Agent dataclass + methods
│   │
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── gateway.py          # unified complete() interface
│   │   ├── health.py           # async health-check loop
│   │   ├── providers/
│   │   │   ├── __init__.py
│   │   │   ├── llamacpp.py     # llama.cpp-server HTTP client
│   │   │   ├── claude.py       # anthropic SDK wrapper
│   │   │   └── openai_.py      # openai SDK wrapper
│   │   └── types.py            # Message, Completion, etc.
│   │
│   ├── filesystem/
│   │   ├── __init__.py
│   │   ├── reader.py           # read-only file access helpers
│   │   └── scripts.py          # save, list, approve, execute scripts
│   │
│   └── utils/
│       ├── __init__.py
│       ├── config.py           # load settings.yaml + .env
│       └── logging.py          # structured logging setup
│
├── scripts/                    # agent-generated scripts land here
│   └── .gitkeep
│
├── jobs/
│   ├── example_setup_daemon    # existing: systemd setup
│   └── example_startup_service # existing: startup loop
│
└── tests/
    ├── test_gateway.py
    ├── test_agents.py
    └── test_router.py
```

---

## 4. Component Contracts

### 4.1 Discord Bot (`src/bot/`)

**Responsibility:** Translate between Discord messages and orchestrator calls.

```python
# client.py — simplified contract
class OpenClawBot(discord.Client):
    """
    Connects to Discord, listens on a single channel.
    Forwards every message to the orchestrator.
    Posts the response back (chunked if >2000 chars).
    """

    ALLOWED_CHANNEL_ID: int  # from .env
    orchestrator: Orchestrator

    async def on_message(self, message: discord.Message) -> None:
        if message.author == self.user:
            return
        if message.channel.id != self.ALLOWED_CHANNEL_ID:
            return

        response = await self.orchestrator.handle(message.content)
        await self.send_response(message.channel, response)
```

Discord message length limit is 2000 characters. `formatting.py` handles splitting and optionally attaching long output as a `.txt` file.

### 4.2 Orchestrator (`src/orchestrator/`)

**Responsibility:** Parse commands, route to the right agent, manage conversation state, enforce permissions.

```python
# router.py — core dispatch
class Orchestrator:
    registry: AgentRegistry
    gateway: LLMGateway
    script_runner: ScriptRunner
    pending_scripts: dict[str, Path]  # id → file, awaiting /approve

    async def handle(self, raw: str) -> str:
        """
        Dispatch table:
          /agent <name> <msg>    → talk to a specific agent
          /newagent <name> <desc>→ create agent via factory
          /agents                → list registered agents
          /kill <name>           → remove an agent
          /run                   → list pending scripts
          /approve <id>          → execute a pending script
          (anything else)        → route to default agent ("chat")
        """
```

**State management:** Conversation history is held in-memory as a list of `{"role": ..., "content": ...}` dicts per agent. It resets on process restart, which is fine — agents are specialists, not long-term memory stores. If persistence matters later, dump to SQLite.

### 4.3 Agent Registry & Factory (`src/agents/`)

**Responsibility:** CRUD for agents. Each agent is a YAML file + runtime state.

```yaml
# config/agents/_defaults.yaml — inherited by all agents
permissions:
  read_filesystem: true
  write_filesystem: false
  execute_scripts: false      # can CREATE scripts; execution needs /approve
  network:
    localhost_only: true       # agents cannot make outbound requests
    allowed_hosts: []          # per-agent overrides go here

max_context_tokens: 8192
conversation_history_limit: 50  # messages before oldest get trimmed
```

```yaml
# config/agents/chat.yaml
name: chat
description: General-purpose assistant. Default target for bare messages.
system_prompt: |
  You are OpenClaw's general assistant running on an Arch Linux PC.
  You can read files when the user provides a path.
  You can write single-file scripts (Python or Bash).
  You CANNOT execute scripts directly — the user must approve execution.
  Keep responses concise. You are talking over Discord.
model_preference: null  # use gateway default priority
permissions: {}         # inherits _defaults.yaml
```

**Factory flow (`/newagent codereviewer Reviews Python code for style and bugs`):**

1. Orchestrator calls `AgentFactory.create("codereviewer", "Reviews Python code for style and bugs")`.
2. Factory writes `config/agents/codereviewer.yaml` with a generated system prompt incorporating the description.
3. Registry hot-loads the new agent. No restart needed.
4. Confirmation sent to Discord.

The factory itself can optionally use the LLM gateway to generate a richer system prompt from the short description, or it can use a simple template. Start with the template; upgrade to LLM-generated prompts once the gateway is proven stable.

### 4.4 LLM Gateway (`src/llm/`)

**Responsibility:** Single `complete()` function the rest of the codebase calls. Handles provider selection, health checks, and fallback.

```python
# gateway.py — contract
class LLMGateway:
    providers: list[LLMProvider]  # ordered by priority
    health: HealthMonitor

    async def complete(
        self,
        messages: list[dict],
        system: str | None = None,
        model: str | None = None,     # force a specific provider
        stream: bool = False,
    ) -> str:
        """
        Try providers in priority order.
        Skip any whose health check is failing.
        Raise if all providers are down.
        """
```

```python
# providers/llamacpp.py — example provider
class LlamaCppProvider:
    base_url: str  # e.g. "http://localhost:8080"

    async def complete(self, messages, system=None) -> str:
        # POST to /completion or /v1/chat/completions
        # llama.cpp-server supports the OpenAI-compatible endpoint
        ...

    async def health_check(self) -> bool:
        # GET /health or just try /v1/models
        ...
```

**Health monitor:** An `asyncio` background task that pings the LAN endpoint every 30 seconds. The cached result is a simple boolean. When the LAN model comes online, it's picked up within 30 seconds automatically.

**Provider priority (default):**

| Priority | Provider | When |
|---|---|---|
| 1 | LAN llama.cpp-server | When healthy |
| 2 | Claude (Anthropic API) | LAN down, key set |
| 3 | OpenAI API | LAN down, Claude down/unset |

Override per-agent by setting `model_preference` in the agent YAML (e.g., force Claude for a code-review agent that needs strong reasoning).

### 4.5 Filesystem & Script Runner (`src/filesystem/`)

**Read access:**

```python
# reader.py
class FileReader:
    """
    Agents request file reads through the orchestrator.
    The orchestrator injects file contents into the LLM context.
    The agent never touches the filesystem directly.
    """
    allowed_roots: list[Path]  # e.g. [Path.home()]

    def read(self, path: str) -> str:
        resolved = Path(path).resolve()
        # Confirm it's under an allowed root
        # Return contents (truncated if very large)
```

**Script lifecycle:**

```
Agent produces code block  →  Orchestrator extracts it
        │
        ▼
scripts/<agent>_<timestamp>.py  saved to disk
        │
        ▼
Discord: "Script saved as `chat_20260306_1423.py`. Reply /approve chat_20260306_1423 to execute."
        │
        ▼
You reply: /approve chat_20260306_1423
        │
        ▼
ScriptRunner executes in subprocess with timeout
        │
        ▼
stdout/stderr sent back to Discord
```

Scripts run as your user (no sandboxing needed for a personal project). A configurable timeout (default 60s) kills runaway processes.

---

## 5. Configuration

### `config/settings.yaml`

```yaml
discord:
  channel_id: 123456789012345678  # your private channel

llm:
  providers:
    - name: lan
      type: llamacpp
      url: "http://localhost:8080"
      health_interval_sec: 30
    - name: claude
      type: anthropic
      # key loaded from .env
    - name: openai
      type: openai
      # key loaded from .env

filesystem:
  allowed_read_roots:
    - "~/"
  scripts_dir: "./scripts"
  script_timeout_sec: 60

agents:
  default: chat
  config_dir: "./config/agents"
```

### `.env`

```bash
DISCORD_BOT_TOKEN=your_token_here
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

---

## 6. Key Considerations

### What "no tools" means in practice

The agents themselves have no tool-use / function-calling capability registered with the LLM. Instead, the **orchestrator** acts as the tool layer: it reads files on behalf of the agent (injecting content into context), and it extracts code blocks from agent output to save as scripts. This keeps the agent's LLM calls simple (just messages in, text out) and avoids relying on tool-calling support that may not exist in the LAN model.

### Catastrophic forgetting and the agent factory

The factory pattern solves this by making agents narrow: a "codereviewer" agent has a system prompt focused entirely on code review, and its conversation history only contains code-review interactions. There's no shared history between agents. If an agent's behavior drifts, you `/kill` it and `/newagent` a fresh one — no retraining, no prompt archaeology.

### LAN model intermittency

The health monitor is the critical piece. Without it, every request when the LAN model is down would hang or error before falling back. The 30-second poll means at most a 30-second window where the first request to a just-stopped LAN server will fail and trigger fallback. That's acceptable for a personal tool. If you want faster detection, drop the interval to 10 seconds — the cost is one lightweight HTTP request every 10s.

### Discord as a command interface

Discord's bot websocket is **outbound-only** from your PC's perspective. Your PC connects to Discord's servers; Discord never connects to you. This is the whole reason for using Discord instead of a tunnel: zero inbound ports, zero attack surface. The tradeoff is latency (your commands go PC → Discord → PC) but for a personal assistant this is negligible.

Message length limit (2000 chars) means long responses need chunking or file attachments. The `formatting.py` module handles this.

### Localhost restriction with per-site overrides

By default, agents cannot trigger any outbound network calls. The orchestrator enforces this: if an agent's output contains a URL or a network request instruction, the orchestrator checks `permissions.network.allowed_hosts`. If the host isn't listed, the request is dropped and the agent is told "network access denied." Overrides are set in the agent's YAML file.

In practice, since agents have no tool-use, network access only matters for the script runner: a script the agent generates might contain `requests.get(...)`. The script runner could optionally parse the script for network calls and warn you before `/approve`, but for a personal project, you'll be reading the script anyway before approving.

### What to build first

Recommended implementation order:

1. **LLM Gateway + health monitor** — get `complete()` working against llama.cpp-server and at least one cloud provider. This is the foundation everything else depends on.
2. **Agent base + registry** — load agents from YAML, maintain conversation history in memory, call the gateway.
3. **Orchestrator (no Discord)** — wire up routing and the script lifecycle using a simple CLI REPL for testing. Prove the command flow works before adding Discord.
4. **Discord bot** — connect the orchestrator to Discord. This is mostly plumbing.
5. **Agent factory** — `/newagent` support. Template-based first, LLM-generated prompts later.
6. **File reader** — inject file contents into agent context.
7. **Script runner** — save, approve, execute flow.

### Dependencies

```
# pyproject.toml [project.dependencies]
discord-py      >= 2.3     # Discord bot
anthropic       >= 0.40    # Claude API
openai          >= 1.50    # OpenAI API
httpx           >= 0.27    # async HTTP for llama.cpp + health checks
pyyaml          >= 6.0     # agent config files
python-dotenv   >= 1.0     # .env loading
```

No heavy frameworks. The entire project should stay under ~1500 lines of Python for the core functionality.

---

## 7. Daemon Setup (Arch Linux)

Replace the existing Pi-oriented systemd scripts with one targeting your Arch PC:

```ini
# /etc/systemd/system/openclaw.service
[Unit]
Description=OpenClaw Discord Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=your_username
WorkingDirectory=/home/your_username/pi-openclaw
ExecStart=/home/your_username/pi-openclaw/.venv/bin/python -m src
Restart=on-failure
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw.service
```

---

## 8. Open Questions for You to Decide

These are choices that don't have a single right answer — they depend on how you want to use the tool day to day:

- **Slash commands vs. prefix commands?** Slash commands (`/agent`, `/newagent`) are cleaner in Discord but require registering with the Discord API. Prefix commands (`!agent`, `!newagent`) work with just message content parsing and are simpler to implement. Start with prefix; migrate to slash later if you want.

- **Conversation persistence across restarts?** Currently designed as ephemeral (memory resets on restart). If you want persistence, a single SQLite file with a `conversations` table (agent_name, role, content, timestamp) is trivial to add.

- **Agent-generated prompt refinement?** The factory can either use a simple template ("You are an agent named {name}. Your job: {description}.") or call the LLM to expand the description into a detailed system prompt. The latter produces better agents but adds a dependency on a working LLM during agent creation.

- **Script language restrictions?** Currently any language the agent produces. You could restrict to Python-only or Bash-only if you want to keep things predictable.

- **llama.cpp-server model swapping?** If you run different models at different times, the gateway could detect which model is loaded (via `/v1/models`) and adjust behavior or inform the agent. Worth implementing only if you actually swap models frequently.