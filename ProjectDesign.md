# OpenClaw — Project Design Document (Simplified)

**Platform:** Arch Linux
**Language:** Python 3.11+ (+ Bash for setup)  
**Status:** Planning Phase  
**Use case:** Personal — single private Discord channel as the sole command interface

---

## 1. High-Level Architecture

Discord is the remote terminal. You send commands from your private channel; your PC receives them via Discord's outbound websocket (no inbound ports, no tunneling). Every message flows through the orchestrator, which picks the right agent and routes LLM calls through a unified gateway that handles the intermittent LAN model transparently.

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

