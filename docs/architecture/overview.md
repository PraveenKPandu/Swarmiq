# Architecture Overview

## 1. Purpose and Vision

Swarmiq is a **multi-agent AI platform** that lets multiple autonomous AI agents cooperate, communicate, and reason together to solve complex problems that a single agent cannot handle alone. The name is a portmanteau of **Swarm** (collective intelligence) and **IQ** (intelligence quotient), reflecting the platform's goal of amplifying intelligence through collaboration.

### Core Problem

Modern AI tasks are increasingly complex and multi-faceted:
- A single agent is context-limited and domain-specific.
- Parallelism and specialisation improve throughput and quality.
- Long-horizon tasks benefit from decomposition and delegation.

Swarmiq addresses these challenges by providing a framework where many small, specialised agents operate as a coordinated swarm.

---

## 2. Design Goals

| Goal | Description |
|---|---|
| **Modularity** | Every subsystem is independently replaceable. Agents, memory backends, and communication transports are all pluggable. |
| **Scalability** | The platform can scale horizontally by adding more agent workers or distributing workloads across multiple nodes. |
| **Observability** | All agent actions, messages, and state transitions are logged, traced, and inspectable. |
| **Extensibility** | New agent types and tools can be registered at runtime without modifying core platform code. |
| **Safety** | Agents operate within defined permission boundaries; a misbehaving agent cannot corrupt global state. |
| **Language-Agnostic Agents** | While the core platform is Python-first, agents can wrap any subprocess or HTTP service. |

---

## 3. Guiding Principles

1. **Separation of Concerns** – Orchestration logic is decoupled from agent implementation logic.
2. **Event-Driven Communication** – Agents communicate asynchronously via a message bus, avoiding tight coupling.
3. **Immutable Task Objects** – Tasks passed between agents are immutable value objects; transformations produce new objects.
4. **Explicit over Implicit** – Agent capabilities, tool access, and resource limits are declared explicitly in manifests.
5. **Fail Fast, Recover Gracefully** – Errors surface immediately; the orchestrator has built-in retry and fallback strategies.

---

## 4. High-Level Architecture

The platform is organised into four logical layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
│         (CLI · REST/WebSocket API · SDK · Web UI)               │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Tasks / Queries
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Orchestration Layer                        │
│    API Gateway · Task Manager · Swarm Orchestrator · Planner    │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Dispatches work
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Agent Layer                             │
│        Agent Pool · Agent Lifecycle · Tool Executor             │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Reads / Writes
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Infrastructure Layer                        │
│      Message Bus · Memory Store · Tool Registry · Logging       │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

#### Client Layer
The entry point for all external interactions. Provides:
- A **REST API** for programmatic access and webhooks.
- A **WebSocket API** for streaming agent progress and intermediate results.
- A **Python SDK** for embedding Swarmiq in other applications.
- A **CLI** for operator and developer use.

#### Orchestration Layer
The "brain" of the platform. Receives a high-level goal, decomposes it into tasks, assigns tasks to agents, tracks progress, and assembles the final result.

#### Agent Layer
The "workers" of the platform. Each agent is an isolated execution unit with:
- A defined **role** (e.g., researcher, coder, critic, summariser).
- Access to a curated **tool set**.
- Its own **short-term memory** (context window).
- Awareness of the shared **long-term memory** store.

#### Infrastructure Layer
Stateful services shared across all agents and orchestration components:
- **Message Bus** – async pub/sub for inter-agent and agent-to-orchestrator communication.
- **Memory Store** – vector + key-value storage for shared knowledge and conversation history.
- **Tool Registry** – catalog of available tools and their capability schemas.
- **Logging & Tracing** – structured event logs for debugging and auditing.

---

## 5. Key Concepts

### Swarm
A named group of agents assembled for a specific goal. The swarm has a lifecycle:
`PENDING → PLANNING → RUNNING → REVIEWING → COMPLETE | FAILED`

### Agent
An autonomous unit that receives a task, reasons about it (potentially calling tools or sub-agents), and produces a result. Agents are stateless workers; all persistent state lives in the infrastructure layer.

### Task
An immutable work item passed between the orchestrator and agents. A task contains:
- A **goal** (natural language description)
- **Context** (relevant prior results, memory snippets)
- **Constraints** (time limit, max cost, allowed tools)
- A **parent task ID** (for sub-task tracing)

### Tool
A capability an agent can invoke – e.g., web search, code execution, file read/write, API call. Tools are registered in the Tool Registry and described by a JSON Schema.

### Memory
| Type | Scope | Backend |
|---|---|---|
| Working Memory | Single agent turn | In-process (context window) |
| Episodic Memory | Swarm lifetime | Vector store (semantic search) |
| Semantic Memory | Global / persistent | Vector store + knowledge graph |
| Procedural Memory | Tool execution history | Key-value store |

---

## 6. Deployment Topology

### Single-Node (Development)
All components run in a single Python process using an in-process message bus and in-memory stores.

```
[CLI / API]──▶[Orchestrator]──▶[Agent Workers (threads)]
                                        │
                              [In-Memory Bus & Store]
```

### Distributed (Production)
Components are deployed as separate services, connected via a message broker (e.g., Redis Streams or Kafka) and a shared vector database.

```
[Load Balancer]
      │
[API Gateway Pods]──▶[Orchestrator Service]
                              │
                  [Agent Worker Pods (auto-scaled)]
                              │
              [Redis Streams (Message Bus)]
              [Qdrant / Weaviate (Memory Store)]
              [PostgreSQL (Task Metadata)]
```

---

## 7. Technology Stack

| Concern | Chosen Technology | Rationale |
|---|---|---|
| Core language | Python 3.11+ | Rich AI/ML ecosystem; async-native |
| Agent reasoning | LLM APIs (OpenAI, Anthropic, local) | Pluggable via a unified adapter |
| Async runtime | `asyncio` / `anyio` | Non-blocking I/O for agent concurrency |
| Message bus (dev) | In-process event emitter | Zero-dependency local development |
| Message bus (prod) | Redis Streams | Low-latency, persistent, widely available |
| Vector memory | Qdrant (default) | Fast ANN search; Docker-friendly |
| Task metadata | SQLite (dev) / PostgreSQL (prod) | ACID guarantees for task state |
| API framework | FastAPI | OpenAPI docs; async; Pydantic models |
| Serialisation | Pydantic v2 | Schema validation and JSON serialisation |
| Packaging | `uv` + `pyproject.toml` | Modern, fast Python packaging |
| Observability | OpenTelemetry | Vendor-neutral traces and metrics |
