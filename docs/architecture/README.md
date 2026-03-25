# Swarmiq – Architecture Documentation

This directory contains the architecture documentation for **Swarmiq**, a multi-agent AI platform designed to coordinate autonomous agents that collaborate, reason, and act as a unified system.

---

## Contents

| Document | Description |
|---|---|
| [Overview](overview.md) | High-level system architecture, design goals, and guiding principles |
| [Components](components.md) | Detailed description of every major component and its responsibilities |
| [Data Flow](data-flow.md) | Request lifecycle, agent communication patterns, and sequence diagrams |
| [Decisions](decisions.md) | Architecture Decision Records (ADRs) capturing key design choices |

---

## Quick Summary

```
┌──────────────────────────────────────────────────────┐
│                     Swarmiq Platform                 │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌────────────────┐  │
│  │  Client  │───▶│   API    │───▶│  Orchestrator  │  │
│  │  (User)  │    │ Gateway  │    │    (Swarm)     │  │
│  └──────────┘    └──────────┘    └───────┬────────┘  │
│                                          │            │
│           ┌──────────────────────────────┤            │
│           │              │               │            │
│    ┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐    │
│    │   Agent A   │ │   Agent B   │ │   Agent C   │    │
│    │ (Specialist)│ │ (Specialist)│ │ (Specialist)│    │
│    └─────────────┘ └────────────┘ └────────────┘    │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │            Shared Infrastructure                │ │
│  │  Memory Store │ Message Bus │ Tool Registry     │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

See [Overview](overview.md) for the full architecture narrative.

---

## How to Read These Docs

1. Start with **[Overview](overview.md)** to understand the big picture and design philosophy.
2. Move to **[Components](components.md)** to learn about each subsystem in depth.
3. Read **[Data Flow](data-flow.md)** to understand how the pieces work together at runtime.
4. Consult **[Decisions](decisions.md)** to understand *why* specific design choices were made.
