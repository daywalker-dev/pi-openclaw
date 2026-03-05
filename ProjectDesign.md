# OpenClaw — Project Design Document

**Platform:** Raspberry Pi  
**Language:** Python (+ Bash for system-level tasks)  
**Status:** Planning Phase

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Discord Interface                     │
│         (commands in, status/responses out)              │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  Orchestrator (Core)                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ Task Router  │  │ Agent Manager │  │ State Manager  │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
└─────────┼────────────────┼──────────────────┼───────────┘
          │                │                  │
┌─────────▼────────────────▼──────────────────▼───────────┐
│                   Agent Registry                         │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │ Agent:Claw │ │ Agent:Chat │ │ Agent:Mod  │  ...      │
│  │ (task-spec)│ │ (task-spec)│ │ (task-spec)│           │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘           │
│        │  isolated     │  isolated    │  isolated        │
│        │  context      │  context     │  context         │
└────────┼───────────────┼──────────────┼─────────────────┘
         │               │              │
┌────────▼───────────────▼──────────────▼─────────────────┐
│                  LLM Router / Gateway                    │
│  ┌──────────────┐ ┌────────────┐ ┌───────────────────┐  │
│  │ LAN LLM      │ │ Claude API │ │ OpenAI API        │  │
│  │ (primary,    │ │ (fallback/ │ │ (fallback/        │  │
│  │  intermittent)│ │  specialty)│ │  specialty)       │  │
│  └──────────────┘ └────────────┘ └───────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure

```
openclaw/
├── main.py                     # Entry point, starts orchestrator + Discord bot
├── config/
│   ├── settings.yaml           # Global config (LLM endpoints, Pi GPIO pins, timeouts)
│   ├── agents.yaml             # Agent definitions and task-type mappings
│   └── secrets.env             # API keys (gitignored)
│
├── core/
│   ├── __init__.py
│   ├── orchestrator.py         # Central coordinator — routes tasks, manages lifecycle
│   ├── task_router.py          # Classifies incoming requests → task type → agent
│   └── state_manager.py        # Persists agent state, conversation context, Pi state
│
├── agents/
│   ├── __init__.py
│   ├── base_agent.py           # Abstract base: system prompt, memory, LLM binding
│   ├── agent_factory.py        # Creates new agents for new task types at runtime
│   ├── agent_registry.py       # Registers, discovers, and retrieves active agents
│   └── builtin/
│       ├── claw_agent.py       # Controls the physical claw via GPIO commands
│       ├── chat_agent.py       # General conversational agent for Discord
│       └── moderation_agent.py # Server moderation / filtering agent
│
├── llm/
│   ├── __init__.py
│   ├── gateway.py              # Unified interface: send prompt → get completion
│   ├── router.py               # Picks backend based on availability + task needs
│   ├── backends/
│   │   ├── lan_llm.py          # Client for your local LAN-hosted model
│   │   ├── claude_api.py       # Anthropic API client
│   │   └── openai_api.py       # OpenAI API client
│   └── health_monitor.py       # Pings LAN LLM, tracks availability windows
│
├── hardware/
│   ├── __init__.py
│   ├── claw_controller.py      # GPIO interface — move, grab, release, home
│   ├── sensor_reader.py        # Any sensors (position, limit switches, current)
│   └── safety.py               # Watchdog timers, current limits, emergency stop
│
├── discord_bot/
│   ├── __init__.py
│   ├── bot.py                  # discord.py bot setup, event loop
│   ├── commands.py             # Slash commands: /claw, /status, /ask, /newagent
│   ├── listeners.py            # Message listeners for passive monitoring
│   └── formatters.py           # Embeds, status cards, response formatting
│
├── memory/
│   ├── __init__.py
│   ├── context_store.py        # Per-agent conversation/context persistence
│   ├── task_memory.py          # Per-task-type long-term knowledge (avoids forgetting)
│   └── vector_store.py         # Optional: lightweight local embeddings for retrieval
│
├── utils/
│   ├── __init__.py
│   ├── logging_config.py       # Structured logging (rotating file + stdout)
│   ├── retry.py                # Exponential backoff for flaky LAN/API calls
│   └── resource_monitor.py     # CPU/RAM/temp monitoring for the Pi
│
├── scripts/
│   ├── setup_pi.sh             # System dependencies, GPIO permissions, venv creation
│   ├── install.sh              # pip install, config scaffolding
│   ├── healthcheck.sh          # Cron-friendly: is the bot alive? Is the LAN LLM up?
│   └── deploy.sh               # Pull, install, restart systemd service
│
├── tests/
│   ├── test_llm_gateway.py
│   ├── test_agent_factory.py
│   ├── test_claw_controller.py # Mock GPIO for CI
│   └── test_discord_commands.py
│
├── systemd/
│   └── openclaw.service        # systemd unit file for autostart on boot
│
├── requirements.txt
├── pyproject.toml
└── README.md
```

---

## 3. Component Design

### 3.1 LLM Router / Gateway

This is the most critical abstraction. Every agent talks to a single `LLMGateway` — it never knows or cares which backend answers.

**Routing logic (priority order):**

1. **LAN LLM** — preferred when available (free, low-latency on LAN, no token cost). The health monitor pings it every N seconds and maintains a `is_available` flag.
2. **Claude API** — fallback for complex reasoning, tool use, or when LAN is down.
3. **OpenAI API** — second fallback, or preferred for specific tasks (e.g., vision, function calling with specific models).

**Key interface:**

```python
class LLMGateway:
    async def complete(
        self,
        messages: list[dict],
        system_prompt: str,
        backend_preference: str | None = None,  # "lan", "claude", "openai", or None for auto
        max_tokens: int = 1024,
        temperature: float = 0.7,
    ) -> CompletionResult:
        """Route to best available backend, return unified result."""
```

**Key considerations:**

- **Intermittent LAN LLM:** The health monitor must be non-blocking. Use an async ping loop that updates a shared flag. The router checks the flag synchronously — never blocks waiting for the LAN model to come online.
- **Graceful degradation:** If all backends are down, queue the request and notify via Discord that the system is in degraded mode.
- **Token budget tracking:** Log token usage per backend per agent. Pi has limited storage, so rotate logs aggressively.
- **Response normalization:** Each backend returns different shapes. The gateway normalizes everything into a single `CompletionResult(text, usage, backend_used, latency_ms)`.

### 3.2 Agent System (Avoiding Catastrophic Forgetting)

The core idea: **each task type gets its own agent with isolated context, system prompt, and memory.** No single monolithic agent tries to do everything.

**Base agent structure:**

```python
class BaseAgent:
    agent_id: str               # unique, e.g. "claw-control-v1"
    task_type: str              # e.g. "claw_operation", "general_chat", "moderation"
    system_prompt: str          # task-specific persona and instructions
    memory: TaskMemory          # isolated long-term memory for this task type
    context_window: list[dict]  # rolling conversation buffer (capped size)
    llm_preference: str | None  # can pin an agent to a specific backend
```

**Agent Factory — creating agents for new task types:**

```python
class AgentFactory:
    def create_agent(
        self,
        task_type: str,
        system_prompt: str,
        llm_preference: str | None = None,
        memory_strategy: str = "rolling_window",  # or "rag", "summarize"
    ) -> BaseAgent:
        """
        Instantiate a new agent with:
        - A fresh, isolated memory store
        - Its own system prompt (no bleed from other agents)
        - Registered in the AgentRegistry for routing
        """
```

**How catastrophic forgetting is avoided:**

| Strategy | How it works |
|---|---|
| **Isolation** | Each agent has its own system prompt and memory. Learning about claw physics never overwrites chat personality. |
| **Persistent task memory** | Each agent's accumulated knowledge is saved to disk (JSON/SQLite) under a namespaced key. On restart, the agent reloads its own history. |
| **No shared weights** | Since we're calling external LLMs (not fine-tuning), "forgetting" means context window overflow, not weight drift. We manage this with summarization and RAG. |
| **Context summarization** | When an agent's context window fills up, older messages are summarized into a condensed "memory block" prepended to the system prompt rather than dropped entirely. |
| **Optional RAG** | For agents that accumulate a lot of knowledge (e.g., a moderation agent learning server-specific rules), a lightweight local vector store (ChromaDB or FAISS with sentence-transformers) enables retrieval without stuffing everything into the prompt. |

**Runtime agent creation via Discord:**

A `/newagent` slash command lets you define a new agent on the fly:
```
/newagent type:recipe_helper prompt:"You are a cooking assistant..." backend:claude
```
This calls `AgentFactory.create_agent()`, registers it, and immediately makes it available for routing.

### 3.3 Hardware Interface (Claw Controller)

```python
class ClawController:
    """All GPIO interaction is isolated here. Nothing else touches pins."""

    def move(self, axis: str, direction: str, duration_ms: int) -> MoveResult
    def grab(self) -> GrabResult
    def release(self) -> None
    def home(self) -> None          # return to known position
    def emergency_stop(self) -> None # kill all motors immediately
    def get_position(self) -> Position
```

**Safety layer** (`safety.py`):

- Watchdog timer: if no command received in N seconds, auto-home.
- Current monitoring: if motor current exceeds threshold, stop and report.
- Boundary limits: software-enforced position limits even if limit switches fail.
- All safety checks run in a separate thread/process, not dependent on the main event loop.

**How agents control hardware:**

The claw agent doesn't call GPIO directly. It emits structured commands (JSON), which the orchestrator validates and forwards to `ClawController`. This keeps the agent sandboxed.

```
Agent output:  {"action": "move", "axis": "x", "direction": "left", "duration_ms": 500}
Orchestrator:  validates schema → calls claw_controller.move("x", "left", 500)
```

### 3.4 Discord Bot Interface

Built with `discord.py` (async, well-maintained, supports slash commands).

**Command flow:**

```
User sends /claw grab
  → bot.py receives interaction
  → commands.py parses intent
  → orchestrator.task_router classifies as "claw_operation"
  → agent_registry finds claw_agent
  → claw_agent calls LLMGateway (if reasoning needed) or acts directly
  → result flows back through orchestrator
  → formatters.py builds an embed
  → bot sends response to channel
```

**Key commands:**

| Command | Description |
|---|---|
| `/claw <action>` | Direct claw control (move, grab, release, home) |
| `/ask <question>` | Route to appropriate agent based on content |
| `/status` | System health: Pi temp/CPU, LLM availability, agent count |
| `/newagent` | Create a new task-specific agent at runtime |
| `/agents` | List all registered agents and their status |
| `/logs <agent>` | Retrieve recent activity for a specific agent |

**Listeners (passive):**

- Monitor specific channels for messages that match patterns (e.g., "hey bot, ..." or @mentions).
- Feed matched messages to the task router just like slash commands.

### 3.5 State Management

All state lives on the Pi's filesystem (SD card or USB SSD).

```
~/.openclaw/
├── state/
│   ├── agents/
│   │   ├── claw-control-v1.json      # agent config + memory snapshot
│   │   ├── general-chat-v1.json
│   │   └── ...
│   ├── llm_usage.json                 # token counts per backend per day
│   └── hardware_state.json            # last known claw position, calibration
├── logs/
│   ├── openclaw.log                   # rotating, 10MB max, 3 files
│   └── discord.log
└── vector_db/                         # optional, for RAG-enabled agents
    └── chroma/
```

---

## 4. Key Interfaces Between Components

```
Discord Bot ──(async message queue)──► Orchestrator
Orchestrator ──(task classification)──► Task Router
Task Router ──(agent lookup)──────────► Agent Registry
Agent Registry ──(dispatch)───────────► Specific Agent
Agent ──(prompt + messages)───────────► LLM Gateway
LLM Gateway ──(health check)─────────► Health Monitor
LLM Gateway ──(API call)─────────────► LAN / Claude / OpenAI
Agent ──(structured command)──────────► Orchestrator ──► Claw Controller
Agent ──(read/write)──────────────────► Task Memory / Context Store
```

All inter-component communication is **async** (asyncio). The Discord bot event loop is the main loop; hardware commands are dispatched to a thread pool to avoid blocking.

---

## 5. Key Considerations

### 5.1 Raspberry Pi Constraints

- **RAM:** Likely 1–8 GB depending on model. Budget carefully. No local model inference — all LLM calls go to LAN or cloud. Keep vector DB small or skip it initially.
- **CPU:** Adequate for orchestration, GPIO, and Discord bot. Not adequate for any ML inference.
- **Storage:** SD cards are slow and wear out. Use a USB SSD for state/logs if possible. Minimize write frequency (batch state saves, don't write every message).
- **Thermal:** Monitor CPU temp via `resource_monitor.py`. Throttle activity if temp exceeds 75°C. The Pi will thermal-throttle on its own at 80°C, but you want to catch it earlier.
- **Network:** Reliable WiFi/Ethernet is essential. The intermittent LAN LLM makes network handling the most likely failure point.

### 5.2 LAN LLM Intermittency

This is the hardest engineering problem in the project.

- **Don't assume availability.** Every call through the gateway must have a timeout (2–5s for health check, 30–60s for completion).
- **Circuit breaker pattern:** After N consecutive failures, stop trying the LAN LLM for M seconds before probing again. This prevents piling up timeouts.
- **Queue + retry:** If a task isn't urgent, queue it for the LAN LLM and retry when it comes back online. Store the queue in SQLite so it survives restarts.
- **Notify on state changes:** When the LAN LLM goes up or down, post a message to a Discord status channel.

### 5.3 Catastrophic Forgetting — Deeper Notes

Since you're calling external APIs (not training/fine-tuning), "forgetting" manifests as:

1. **Context window overflow** — old context gets truncated, losing learned information. Mitigate with summarization and RAG.
2. **Prompt drift** — if a single agent accumulates too many responsibilities, its system prompt becomes incoherent. Mitigate by keeping agents narrowly scoped.
3. **Cross-task contamination** — if agents share memory, claw-control knowledge could pollute chat responses. Mitigate with strict isolation.

The agent factory pattern solves all three: new task, new agent, fresh context, isolated memory.

### 5.4 Security

- **API keys** in `secrets.env`, loaded via `python-dotenv`, never logged, never sent to Discord.
- **Discord permissions:** The bot should request minimal permissions. Claw commands should be restricted to specific roles.
- **Hardware sandboxing:** Agents never touch GPIO directly. The orchestrator validates every hardware command against a schema before forwarding.
- **Rate limiting:** Cap the number of LLM calls per agent per minute to prevent runaway loops.

### 5.5 Deployment & Operations

- **systemd service** for auto-start on boot and auto-restart on crash.
- **`healthcheck.sh`** run via cron every 5 minutes — checks if the process is alive, if Discord is connected, if the LAN LLM is reachable. Sends a Discord webhook alert on failure.
- **Rolling updates:** `deploy.sh` does `git pull`, `pip install -r requirements.txt`, `systemctl restart openclaw`. Keep it simple.

---

## 6. Suggested Build Order

| Phase | What to build | Why this order |
|---|---|---|
| **1** | `llm/gateway.py`, `llm/router.py`, `llm/backends/lan_llm.py` | Get a single LLM call working end-to-end. This unblocks everything else. |
| **2** | `agents/base_agent.py`, `agents/agent_factory.py`, `agents/agent_registry.py` | Establish the agent pattern before building specific agents. |
| **3** | `core/orchestrator.py`, `core/task_router.py` | Wire agents to the gateway through a central coordinator. |
| **4** | `agents/builtin/chat_agent.py` + basic tests | A simple chat agent proves the full pipeline works. |
| **5** | `discord_bot/bot.py`, `discord_bot/commands.py` | Now you have a working bot you can talk to on Discord. |
| **6** | `hardware/claw_controller.py`, `hardware/safety.py` | Add physical claw control with safety layer. |
| **7** | `agents/builtin/claw_agent.py` | The claw agent can now reason about and issue hardware commands. |
| **8** | `llm/backends/claude_api.py`, `llm/backends/openai_api.py` | Add cloud fallbacks. |
| **9** | `memory/context_store.py`, `memory/task_memory.py` | Persistent memory so agents survive restarts. |
| **10** | `llm/health_monitor.py`, `scripts/healthcheck.sh`, systemd | Production hardening. |

---

## 7. Dependencies (Initial)

```
# Core
asyncio             # stdlib
pyyaml              # config parsing
python-dotenv       # secrets loading

# Discord
discord.py          # bot framework (v2.x for slash commands)

# LLM Clients
anthropic           # Claude API
openai              # OpenAI API
httpx               # async HTTP for LAN LLM + health checks

# Hardware
RPi.GPIO            # GPIO control (or gpiozero for a friendlier API)
# lgpio             # alternative if running on Pi 5 (RPi.GPIO has issues)

# State / Memory
aiosqlite           # async SQLite for state persistence
chromadb            # optional, for RAG vector store

# Utilities
structlog           # structured logging
tenacity            # retry logic
psutil              # system resource monitoring
```

---

## 8. Open Questions for Next Phase

1. **Which LAN LLM?** (Ollama? llama.cpp server? vLLM?) — determines the client protocol in `lan_llm.py`.
2. **Claw hardware specifics** — stepper vs servo motors, how many axes, what driver board?
3. **Pi model** — Pi 4 vs Pi 5 affects GPIO library choice and available RAM.
4. **Discord server structure** — dedicated channels for claw control vs general chat vs status?
5. **Token budget** — monthly spend cap for Claude/OpenAI APIs? This affects routing aggressiveness toward the LAN LLM.
6. **RAG necessity** — is lightweight context summarization enough initially, or do agents need retrieval from day one?