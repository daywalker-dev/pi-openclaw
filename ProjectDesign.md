# OpenClaw — Project Design Document (Simplified)

**Platform:** Raspberry Pi 4  
**Language:** Python 3.11+ (+ Bash for setup)  
**Status:** Planning Phase  
**Use case:** Personal — single private Discord channel as the sole command interface

---

## 1. High-Level Architecture

Discord is the remote terminal. You send commands from your private channel; the Pi receives them via Discord's outbound websocket (no inbound ports, no tunneling). Every message flows through the orchestrator, which picks the right agent and routes LLM calls through a unified gateway that handles the intermittent LAN model transparently.

```
┌──────────────────────────────────────────────────────┐
│           Private Discord Channel (you)               │
│        commands in ──── responses/status out           │
└────────────────────────┬─────────────────────────────┘
                         │  outbound websocket only
┌────────────────────────▼─────────────────────────────┐
│                    Orchestrator                        │
│         ┌──────────────┐  ┌──────────────┐            │
│         │ Task Router   │  │ State Manager │            │
│         └──────┬───────┘  └──────┬───────┘            │
└────────────────┼─────────────────┼───────────────────┘
                 │                 │
┌────────────────▼─────────────────▼───────────────────┐
│                  Agent Registry                       │
│    ┌────────────┐  ┌────────────┐                     │
│    │ Agent:Claw │  │ Agent:Chat │   ...more via       │
│    │ (isolated) │  │ (isolated) │   /newagent          │
│    └─────┬──────┘  └─────┬──────┘                     │
└──────────┼───────────────┼───────────────────────────┘
           │               │
┌──────────▼───────────────▼───────────────────────────┐
│               LLM Gateway + Health Monitor            │
│    ┌──────────────┐  ┌──────────┐  ┌──────────────┐  │
│    │ LAN LLM      │  │ Claude   │  │ OpenAI       │  │
│    │ (preferred)   │  │ (fallback)│  │ (fallback)   │  │
│    └──────────────┘  └──────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────┘
           │
┌──────────▼───────────────────────────────────────────┐
│              Hardware (Claw Controller)                │
│              GPIO via RPi.GPIO / gpiozero             │
└──────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure

Flattened from the previous version. The moderation agent, safety watchdog, sensor reader, and resource monitor are gone — you're standing next to the claw and it's your Pi.

```
openclaw/
├── main.py                     # Entry point — starts Discord bot + orchestrator
├── config/
│   ├── settings.yaml           # LLM endpoints, GPIO pins, timeouts, agent defaults
│   ├── agents.yaml             # Builtin agent definitions + any runtime-created ones
│   └── secrets.env             # API keys (gitignored)
│
├── core/
│   ├── __init__.py
│   ├── orchestrator.py         # Routes tasks, manages agent lifecycle
│   ├── task_router.py          # Message → task type → agent lookup
│   └── state_manager.py        # Persists agent state + context to disk
│
├── agents/
│   ├── __init__.py
│   ├── base_agent.py           # Abstract base: system prompt, memory, LLM binding
│   ├── agent_factory.py        # Creates new agents at runtime
│   ├── agent_registry.py       # Tracks all active agents
│   └── builtin/
│       ├── claw_agent.py       # Emits structured GPIO commands
│       └── chat_agent.py       # General-purpose conversational agent
│
├── llm/
│   ├── __init__.py
│   ├── gateway.py              # Unified async interface — agents call this, never backends
│   ├── router.py               # Picks backend: LAN → Claude → OpenAI (with circuit breaker)
│   ├── backends/
│   │   ├── lan_llm.py          # httpx client for local model (Ollama/llama.cpp/etc.)
│   │   ├── claude_api.py       # Anthropic SDK wrapper
│   │   └── openai_api.py       # OpenAI SDK wrapper
│   └── health_monitor.py       # Async ping loop for LAN LLM availability
│
├── hardware/
│   ├── __init__.py
│   └── claw_controller.py      # All GPIO in one place — move, grab, release, home
│
├── discord_bot/
│   ├── __init__.py
│   ├── bot.py                  # discord.py client, event loop, message dispatch
│   └── commands.py             # Slash commands + plain-message parsing
│
├── memory/
│   ├── __init__.py
│   ├── context_store.py        # Per-agent rolling conversation buffer
│   └── task_memory.py          # Per-agent persistent knowledge (JSON/SQLite)
│
├── utils/
│   ├── __init__.py
│   ├── logging_config.py       # Rotating file logger (stdlib logging, no extra deps)
│   └── retry.py                # Tenacity wrappers for LLM calls
│
├── scripts/
│   ├── setup.sh                # venv, pip install, config scaffolding (distro-agnostic)
│   └── deploy.sh               # git pull + restart service
│
├── tests/
│   ├── test_llm_gateway.py
│   ├── test_agent_factory.py
│   └── test_claw_controller.py
│
├── systemd/
│   └── openclaw.service        # Auto-start on boot (optional — see note on portability)
│
├── requirements.txt
└── README.md
```

**What got cut and why:**

| Removed | Reason |
|---|---|
| `moderation_agent.py` | Private channel, single user — no moderation needed |
| `safety.py`, `sensor_reader.py` | You're physically present; hardware monitoring is overhead |
| `resource_monitor.py` | Pi 4 thermals are fine for this workload without active monitoring |
| `listeners.py`, `formatters.py` | Single channel — commands.py handles everything; embeds are nice-to-have, not a separate file |
| `healthcheck.sh` | Folded into a `/status` command — you'll notice if the bot goes offline |
| `vector_store.py` | Deferred — context summarization is enough initially; add RAG later if needed |

---

## 3. Component Design

### 3.1 LLM Gateway + Health Monitor

Every agent calls `gateway.complete()`. It never knows which backend answers.

```python
@dataclass
class CompletionResult:
    text: str
    backend_used: str       # "lan", "claude", "openai"
    latency_ms: float
    usage: dict | None      # token counts if the backend reports them

class LLMGateway:
    async def complete(
        self,
        messages: list[dict],
        system_prompt: str,
        backend_preference: str | None = None,  # override auto-routing
        max_tokens: int = 1024,
        temperature: float = 0.7,
    ) -> CompletionResult:
        ...
```

**Routing priority:** LAN LLM → Claude → OpenAI. The router checks `health_monitor.is_available` before attempting the LAN backend.

**Circuit breaker for LAN intermittency:** After 3 consecutive failures (timeout or connection refused), the router marks the LAN LLM as down for 60 seconds, then sends a single probe. This avoids stacking 30-second timeouts on every request while the LAN host is asleep. When the LAN LLM comes back, the bot posts a short notice to your Discord channel.

**Timeouts:** Health check ping: 3s. Completion request: 45s for LAN, 30s for cloud APIs.

**Response normalization:** Each backend returns a different shape (Ollama returns `{"response": ...}`, Anthropic returns `{"content": [...]}`). The backend wrappers in `backends/` each implement the same `async def complete(...)` signature and return a uniform `CompletionResult`. The gateway never parses raw API responses.

### 3.2 Agent System

**The core anti-forgetting pattern: one task type → one agent → isolated everything.**

```python
class BaseAgent(ABC):
    agent_id: str               # e.g. "claw-v1", "chat-v1", "recipe-helper"
    task_type: str              # routing key
    system_prompt: str          # what this agent is and does
    memory: TaskMemory          # persistent, namespaced to this agent
    context: list[dict]         # rolling window of recent messages
    llm_preference: str | None  # pin to a backend if needed

    async def handle(self, message: str, context: dict) -> str:
        """Process a routed message and return a response."""
        ...

    def summarize_context(self) -> str:
        """Compress old context into a memory block when window fills up."""
        ...
```

**Agent Factory:**

```python
class AgentFactory:
    def create(
        self,
        task_type: str,
        system_prompt: str,
        llm_preference: str | None = None,
        context_limit: int = 50,     # max messages before summarization
    ) -> BaseAgent:
        """
        1. Instantiate agent with fresh isolated memory
        2. Register it in the AgentRegistry
        3. Persist config to agents.yaml so it survives restarts
        """
```

Called by the `/newagent` command:
```
/newagent type:recipe_helper prompt:You are a cooking assistant that gives concise recipes.
```

**How forgetting is avoided:**

The three failure modes and their mitigations:

1. **Context window overflow** — When an agent's rolling context hits its limit, older exchanges are summarized by the LLM into a compressed "memory block" that gets prepended to the system prompt. Raw messages are dropped, but the knowledge is retained in summary form.

2. **Prompt drift** — Each agent has a single, narrow system prompt that never changes. A claw agent never gets asked about recipes. If a new kind of task appears, it gets a new agent, not new instructions bolted onto an existing one.

3. **Cross-task contamination** — Agent memory is namespaced. `~/.openclaw/state/agents/claw-v1.json` and `chat-v1.json` are completely separate files. No shared context, no shared memory, no bleed.

### 3.3 Hardware Interface

Simplified to a single file. The claw agent emits structured JSON commands; the orchestrator forwards them to `ClawController`.

```python
class ClawController:
    """All GPIO access lives here. Nothing else imports RPi.GPIO."""

    def __init__(self, pin_config: dict):
        # pin_config loaded from settings.yaml
        ...

    def move(self, axis: str, direction: str, duration_ms: int) -> dict
    def grab(self) -> dict
    def release(self) -> dict
    def home(self) -> dict
    def emergency_stop(self) -> dict
```

**Agent → Hardware flow:**
```
claw_agent returns: {"action": "move", "axis": "x", "direction": "left", "duration_ms": 500}
orchestrator validates action exists → calls claw_controller.move("x", "left", 500)
result dict sent back to Discord
```

The agent never imports GPIO libraries. If you're testing on a non-Pi machine, the controller can be swapped for a mock that just logs commands — the rest of the system doesn't notice.

### 3.4 Discord Bot

Single private channel. The bot connects outbound to Discord's gateway (websocket) — no ports opened on the Pi. You send messages; the bot receives them, routes through the orchestrator, and replies in the same channel.

**Message handling** — Two paths, both handled in `commands.py`:

1. **Slash commands** — structured, easy to parse: `/claw grab`, `/ask what is X`, `/newagent ...`, `/status`
2. **Plain messages** — the bot listens for any message in the channel (or @mentions) and feeds it to the task router, which uses a simple keyword/regex match (or optionally an LLM classification call) to pick the right agent.

**Commands:**

| Command | What it does |
|---|---|
| `/claw <action>` | Move, grab, release, home |
| `/ask <question>` | Route to best-matching agent |
| `/newagent type:<t> prompt:<p>` | Create and register a new agent |
| `/agents` | List registered agents |
| `/status` | LLM backend availability, uptime, agent count |

**Why discord.py:** It's async-native, supports slash commands in v2.x, and handles the websocket lifecycle for you. The bot's event loop is the application's main loop; everything else (LLM calls, GPIO) is dispatched as async tasks or thread pool calls within it.

### 3.5 State Management

All state is flat files under `~/.openclaw/`. SQLite is used only for task memory (structured queries), everything else is JSON or YAML.

```
~/.openclaw/
├── state/
│   ├── agents/
│   │   ├── claw-v1.json          # agent config + summarized memory
│   │   ├── chat-v1.json
│   │   └── recipe-helper.json    # runtime-created agent
│   └── llm_usage.json            # token counts per backend (rotated daily)
└── logs/
    └── openclaw.log              # single rotating log, 5MB × 3 files
```

State is saved on a debounced timer (every 60s if dirty) rather than on every message. This reduces SD card writes. If you add a USB drive later, just change the base path in `settings.yaml`.

---

## 4. Data Flow

```
You (Discord) ──msg──► bot.py ──► orchestrator.task_router
                                        │
                              ┌─────────▼──────────┐
                              │  agent_registry     │
                              │  find by task_type  │
                              └─────────┬──────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  matched agent      │
                              │  (e.g. claw_agent)  │
                              └─────────┬──────────┘
                                        │
                    ┌───────────────────┤
                    ▼                    ▼
            LLM Gateway           ClawController
            (if reasoning          (if hardware
             needed)                action needed)
                    │                    │
                    ▼                    ▼
            CompletionResult        MoveResult / etc.
                    │                    │
                    └────────┬───────────┘
                             ▼
                    orchestrator combines
                    response + result
                             │
                             ▼
                    bot.py sends reply
                    to Discord channel
```

All async. GPIO calls are wrapped in `asyncio.to_thread()` since RPi.GPIO is synchronous.

---

## 5. Key Considerations

### 5.1 Pi 4 Specifics

The Pi 4 has 1–8 GB RAM and a BCM2711 SoC. For this workload (no local inference, just orchestration + GPIO + network I/O), even the 2 GB model is fine. Key points:

- **GPIO library:** `RPi.GPIO` works well on Pi 4. `gpiozero` is a friendlier wrapper if you prefer. Both are available via pip and don't tie you to a specific distro.
- **Python:** Use 3.11+ for `asyncio.TaskGroup` and performance improvements. Install via your distro's package manager or `pyenv` if the system Python is too old.
- **SD card wear:** Debounce writes (see state management above). If the Pi runs 24/7, consider a USB SSD for `~/.openclaw/` — SD cards degrade over months of continuous logging.

### 5.2 Distro Agnosticism

The project avoids distro-specific assumptions where practical:

| Concern | Approach |
|---|---|
| **Python** | Use a venv. `setup.sh` checks for `python3` ≥ 3.11 and creates one. |
| **GPIO** | `RPi.GPIO` / `gpiozero` are pip packages, not distro-specific. |
| **Service management** | `systemd/openclaw.service` is provided since virtually all Pi distros use systemd (Raspberry Pi OS, Ubuntu, DietPi, Armbian). If someone doesn't have systemd, they can run `main.py` in a `tmux`/`screen` session instead. |
| **Dependencies** | `setup.sh` installs via pip inside the venv. System-level deps (if any) are called out with both `apt` and generic names so you can adapt. |
| **Config paths** | `~/.openclaw/` uses `$HOME`, not a hardcoded path. |

The one unavoidable Pi-specific tie is the GPIO library. If you ever port this to a non-Pi SBC, you'd swap `claw_controller.py` and its imports — the rest of the system is unaffected.

### 5.3 LAN LLM Intermittency

This remains the trickiest part of the project. The design handles it at three levels:

1. **Health monitor** — A background async task pings the LAN endpoint every 15 seconds. Maintains a simple boolean `is_available` flag plus a `last_seen` timestamp.

2. **Circuit breaker in the router** — 3 failures → mark down for 60s → probe → repeat. Prevents every user command from eating a 45-second timeout when the LAN host is off.

3. **Fallback** — When the LAN model is unavailable, requests automatically go to Claude or OpenAI. The `CompletionResult.backend_used` field lets you (or the agent) know which backend answered, if that ever matters.

No queuing for now — if you send a command and the LAN is down, it falls back immediately. Queuing adds complexity that isn't needed when cloud fallbacks exist.

### 5.4 Catastrophic Forgetting — Summary

Since all LLM calls are stateless API requests (no fine-tuning, no local weights), "forgetting" is purely a context management problem. The agent factory pattern solves it:

- **New task → new agent.** Never overload one agent with unrelated responsibilities.
- **Isolated memory.** Each agent's context and persistent memory are namespaced on disk.
- **Summarization over truncation.** When context fills up, compress old exchanges into a summary rather than dropping them.

If an agent eventually needs to recall large amounts of historical knowledge (e.g., hundreds of past claw operations), that's when you'd add a vector store (ChromaDB, FAISS) as a future enhancement. For now, rolling context + summarization is enough.

---

## 6. Build Order

| Phase | Build | Proves |
|---|---|---|
| **1** | `llm/gateway.py` + `llm/backends/lan_llm.py` | You can call the LAN LLM from Python and get a response back. |
| **2** | `llm/router.py` + `llm/health_monitor.py` | Automatic failover works — kill the LAN model mid-conversation and cloud picks up. |
| **3** | `agents/base_agent.py` + `agent_factory.py` + `agent_registry.py` | You can create, register, and dispatch to isolated agents. |
| **4** | `agents/builtin/chat_agent.py` | Full pipeline: message in → agent → LLM → response out (tested locally, no Discord yet). |
| **5** | `discord_bot/bot.py` + `commands.py` | Bot connects, receives commands, replies. You can `/ask` and get a response. |
| **6** | `core/orchestrator.py` + `task_router.py` | Multi-agent routing works — different messages hit different agents. |
| **7** | `hardware/claw_controller.py` + `agents/builtin/claw_agent.py` | `/claw grab` makes the physical claw move. |
| **8** | `memory/context_store.py` + `task_memory.py` + `state_manager.py` | Agents survive restarts with their memory intact. |
| **9** | `llm/backends/claude_api.py` + `openai_api.py` | Cloud fallbacks wired up. |
| **10** | `scripts/setup.sh`, `systemd/openclaw.service`, final cleanup | Runs unattended on boot. |

---

## 7. Dependencies

```txt
# Core
pyyaml              # config files
python-dotenv       # load secrets.env

# Discord
discord.py>=2.0     # async bot with slash command support

# LLM clients
httpx               # async HTTP for LAN LLM (Ollama, llama.cpp, etc.)
anthropic           # Claude API (install when needed)
openai              # OpenAI API (install when needed)

# Hardware
RPi.GPIO            # GPIO on Pi 4 (or gpiozero as alternative)

# State
aiosqlite           # async SQLite for task memory

# Utilities
tenacity            # retry/backoff for flaky calls
```

Kept deliberately small. `structlog`, `psutil`, and `chromadb` from the previous version are dropped — stdlib `logging` is fine, you don't need process monitoring, and RAG is deferred.

---

## 8. Open Questions

1. **Which LAN LLM server?** Ollama exposes an OpenAI-compatible `/v1/chat/completions` endpoint, which would let `lan_llm.py` reuse the same request shape as `openai_api.py`. llama.cpp's server has a slightly different API. This choice affects how much code goes into the LAN backend.
2. **Claw hardware** — stepper vs servo, number of axes, driver board. This determines the `ClawController` implementation.
3. **Token budget** — any monthly cap on Claude/OpenAI spend? If so, the router should track cumulative usage and refuse cloud calls past a threshold.
4. **Context window strategy** — how aggressive should summarization be? Summarize after 20 messages? 50? Should the summarization call itself use the LAN LLM or a cloud model?