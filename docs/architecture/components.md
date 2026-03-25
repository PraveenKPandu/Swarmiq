# Components

This document describes every major component in the Swarmiq platform, its responsibilities, its public interface, and how it relates to other components.

---

## Component Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Swarmiq Platform                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                        Client Layer                          │   │
│  │   ┌──────────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐  │   │
│  │   │  REST /  │  │ WebSocket  │  │ Python   │  │   CLI    │  │   │
│  │   │  HTTP    │  │  Streaming │  │   SDK    │  │          │  │   │
│  │   └────┬─────┘  └─────┬──────┘  └────┬─────┘  └────┬─────┘  │   │
│  └────────┼──────────────┼──────────────┼──────────────┼────────┘   │
│           └──────────────┴──────────────┴──────────────┘           │
│                                    │                                │
│  ┌─────────────────────────────────▼────────────────────────────┐   │
│  │                      Orchestration Layer                     │   │
│  │  ┌─────────────┐  ┌────────────────┐  ┌───────────────────┐  │   │
│  │  │ API Gateway │  │  Swarm         │  │  Task Manager     │  │   │
│  │  │             │  │  Orchestrator  │  │                   │  │   │
│  │  └──────┬──────┘  └───────┬────────┘  └────────┬──────────┘  │   │
│  │         │                 │                    │             │   │
│  │         │          ┌──────▼──────┐             │             │   │
│  │         │          │   Planner   │             │             │   │
│  │         │          └─────────────┘             │             │   │
│  └─────────┼─────────────────────────────────────┼─────────────┘   │
│            │                                     │                  │
│  ┌─────────▼─────────────────────────────────────▼─────────────┐   │
│  │                         Agent Layer                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │   │
│  │  │  Agent Pool  │  │ Agent Runner │  │  Tool Executor    │  │   │
│  │  │              │  │              │  │                   │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      Infrastructure Layer                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │   │
│  │  │ Message Bus  │  │ Memory Store │  │  Tool Registry    │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────────┘  │   │
│  │  ┌──────────────┐  ┌──────────────┐                         │   │
│  │  │ Config &     │  │  Observ-     │                         │   │
│  │  │ Secrets      │  │  ability     │                         │   │
│  │  └──────────────┘  └──────────────┘                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Client Layer

### 1.1 REST / HTTP API

| Property | Value |
|---|---|
| Technology | FastAPI |
| Protocol | HTTP/1.1 and HTTP/2 |
| Auth | API key (Bearer token) or OAuth 2.0 |
| Docs | Auto-generated OpenAPI 3.1 spec at `/docs` |

**Responsibilities**
- Accept task submission requests from external callers.
- Return synchronous responses for short-lived operations (status checks, configuration reads).
- Delegate long-running work to the Orchestration Layer.

**Key Endpoints** (planned)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/swarms` | Create and start a new swarm run |
| `GET` | `/v1/swarms/{id}` | Get swarm status and result |
| `DELETE` | `/v1/swarms/{id}` | Cancel a running swarm |
| `GET` | `/v1/swarms/{id}/events` | Stream events (SSE fallback) |
| `GET` | `/v1/agents` | List registered agent types |
| `GET` | `/v1/tools` | List available tools |
| `POST` | `/v1/memory/search` | Semantic search over memory |

---

### 1.2 WebSocket Streaming API

Provides real-time bidirectional communication for:
- Streaming partial results and intermediate agent thoughts.
- Receiving live task progress events.
- Interactive human-in-the-loop feedback during a swarm run.

**Event Types**

| Event | Direction | Payload |
|---|---|---|
| `task.created` | Server → Client | Task ID, goal |
| `agent.thinking` | Server → Client | Agent ID, reasoning trace |
| `agent.tool_call` | Server → Client | Tool name, arguments |
| `agent.result` | Server → Client | Agent ID, result text |
| `swarm.complete` | Server → Client | Final answer, token usage |
| `human.feedback` | Client → Server | Correction or approval |

---

### 1.3 Python SDK

A thin wrapper around the HTTP and WebSocket APIs that provides:
- Typed `Swarm`, `Agent`, and `Task` model classes (backed by Pydantic).
- Async and sync client variants.
- Automatic retry with exponential back-off.
- Local in-process mode (no server required) for unit-testing integrations.

---

### 1.4 CLI

A developer-facing command-line interface for:
- Running swarms interactively.
- Inspecting task history and memory.
- Managing configuration and secrets.
- Starting the local development server.

---

## 2. Orchestration Layer

### 2.1 API Gateway

**Responsibilities**
- Authentication and authorisation checks.
- Request rate limiting and quota enforcement.
- Request routing to the Swarm Orchestrator or Task Manager.
- Request/response logging for auditability.

The API Gateway is the only public-facing component in the Orchestration Layer; all other components are internal.

---

### 2.2 Swarm Orchestrator

The central coordinator of the platform.

**Responsibilities**
- Own the full lifecycle of a swarm run: `PENDING → PLANNING → RUNNING → REVIEWING → COMPLETE`.
- Invoke the Planner to decompose the top-level goal into a task graph.
- Submit tasks to the Task Manager and track dependencies.
- Handle agent failures: retry, reassign, or escalate.
- Assemble partial results into a coherent final answer.
- Emit swarm lifecycle events to the Message Bus.

**State Machine**

```
 PENDING ──▶ PLANNING ──▶ RUNNING ──▶ REVIEWING ──▶ COMPLETE
               │              │                         ▲
               └──────────────┴──────── FAILED ─────────┘
                         (on unrecoverable error)
```

---

### 2.3 Planner

**Responsibilities**
- Accept a high-level goal and produce a **task graph** (a DAG of interdependent subtasks).
- Select appropriate agent types for each subtask based on their declared capabilities.
- Estimate resource requirements (token budget, time limit) per task.
- Support both static (pre-computed) and dynamic (adaptive) planning:
  - *Static*: the full plan is produced upfront before any agent runs.
  - *Dynamic*: the plan is revised at runtime based on intermediate results.

**Planning Strategies**

| Strategy | Description | Use Case |
|---|---|---|
| Sequential | Tasks executed one after another | Simple, linear workflows |
| Parallel | Independent tasks executed concurrently | Research + summarisation |
| Hierarchical | Sub-swarms for sub-tasks | Complex, multi-domain goals |
| Iterative | Critic loop: Generate → Review → Refine | High-quality output tasks |

---

### 2.4 Task Manager

**Responsibilities**
- Persist task records (goal, status, result, metadata) in the task store.
- Manage task dependencies: a task becomes `READY` when all its predecessors are `COMPLETE`.
- Expose a queue of `READY` tasks that the Agent Pool can consume.
- Update task state as agents report progress.
- Surface task history and lineage for debugging.

**Task States**

```
PENDING ──▶ READY ──▶ ASSIGNED ──▶ RUNNING ──▶ COMPLETE
                                       │
                                    FAILED ──▶ RETRYING ──▶ ABANDONED
```

---

## 3. Agent Layer

### 3.1 Agent Pool

**Responsibilities**
- Maintain a registry of available agent types and their manifests (capability declarations).
- Spawn and recycle agent worker instances.
- Implement a work-stealing queue: idle agents pull tasks from the Task Manager.
- Enforce concurrency limits per agent type.
- Monitor agent health and restart crashed workers.

**Agent Manifest (schema)**

```yaml
name: researcher
description: Searches the web and synthesises information
capabilities:
  - web_search
  - document_summarisation
tools:
  - web_search
  - url_fetch
  - text_summarise
max_concurrent_tasks: 4
model: gpt-4o          # LLM backend (pluggable)
temperature: 0.2
max_tokens: 4096
```

---

### 3.2 Agent Runner

The runtime environment for a single agent execution.

**Responsibilities**
- Load the assigned task and relevant context from Memory Store.
- Construct the agent's prompt (system message + task context + tool definitions).
- Execute the ReAct (Reason + Act) loop:
  1. **Think**: call the LLM to reason about the next step.
  2. **Act**: invoke a tool if the LLM requests one (via function calling / tool use).
  3. **Observe**: inject the tool result back into context.
  4. **Repeat** until the LLM produces a final answer or the turn limit is reached.
- Write the result back to the Task Manager.
- Store important findings in the Memory Store (episodic memory).
- Publish agent lifecycle events to the Message Bus.

**ReAct Loop**

```
┌──────────────────────────────────────────────────────┐
│                    Agent Runner                      │
│                                                      │
│  Task + Context ──▶ Prompt Builder ──▶ LLM Call      │
│                                           │          │
│                             ┌─────────────▼──────┐  │
│                             │  Tool call?         │  │
│                             │  YES ──▶ Tool Exec  │  │
│                             │  NO  ──▶ Final Ans  │  │
│                             └────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

### 3.3 Tool Executor

**Responsibilities**
- Receive a tool invocation request from the Agent Runner.
- Validate arguments against the tool's JSON Schema.
- Execute the tool in an isolated sub-process or sandboxed environment.
- Return the result (or a structured error) to the Agent Runner.
- Enforce per-tool timeout and output size limits.

**Built-in Tool Categories** (planned)

| Category | Examples |
|---|---|
| Information retrieval | `web_search`, `url_fetch`, `knowledge_base_query` |
| Code execution | `python_exec`, `bash_exec` (sandboxed) |
| File system | `file_read`, `file_write`, `directory_list` |
| Communication | `email_send`, `slack_post` |
| Data processing | `json_parse`, `csv_read`, `data_transform` |
| Agent delegation | `spawn_sub_agent`, `query_agent` |

---

## 4. Infrastructure Layer

### 4.1 Message Bus

Provides **asynchronous, decoupled** communication between all components.

| Mode | Backend | When Used |
|---|---|---|
| Development | In-process event emitter | Local runs, unit tests |
| Production | Redis Streams | Distributed deployments |

**Topics / Channels**

| Topic | Publisher | Subscribers |
|---|---|---|
| `swarm.events` | Swarm Orchestrator | Client WebSocket handler, Observability |
| `task.ready` | Task Manager | Agent Pool |
| `task.result` | Agent Runner | Task Manager, Swarm Orchestrator |
| `agent.events` | Agent Runner | Observability, Human-in-the-loop handler |
| `tool.requests` | Agent Runner | Tool Executor |

---

### 4.2 Memory Store

Provides **layered memory** for agents and the orchestrator.

| Memory Type | Backend | Interface | TTL |
|---|---|---|---|
| Working | In-process dict | Read/Write | Single turn |
| Episodic | Vector DB (Qdrant) | Semantic search | Swarm lifetime |
| Semantic | Vector DB + KV | Semantic search + key lookup | Persistent |
| Procedural | Key-Value (Redis) | Key lookup | Configurable |

**Operations**

- `memory.store(content, metadata, scope)` – embed and index a piece of content.
- `memory.search(query, scope, top_k)` – semantic nearest-neighbour search.
- `memory.get(key)` – exact key lookup.
- `memory.delete(key_or_query)` – evict a memory entry.

---

### 4.3 Tool Registry

A catalogue of all tools available to agents.

**Responsibilities**
- Store tool metadata: name, description, input/output JSON Schema.
- Resolve tool names to executable implementations.
- Support dynamic registration of new tools at runtime.
- Enforce tool access control: which agent types can use which tools.

---

### 4.4 Configuration and Secrets

- All configuration is read from environment variables or a `.env` file (never hard-coded).
- Secrets (API keys for LLMs, external services) are loaded from environment variables or a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager).
- Configuration is validated at startup using a Pydantic `Settings` model.

---

### 4.5 Observability

Provides end-to-end visibility into platform behaviour.

| Signal | Technology | Storage |
|---|---|---|
| Structured logs | `structlog` | stdout / log aggregator |
| Distributed traces | OpenTelemetry | Jaeger / OTLP-compatible backend |
| Metrics | OpenTelemetry Metrics | Prometheus / OTLP-compatible backend |

**Key Metrics**

- `swarm.duration_seconds` – histogram of swarm wall-clock time.
- `agent.task_duration_seconds` – per-agent task completion time.
- `agent.llm_tokens_total` – counter of tokens consumed per model.
- `tool.calls_total` – counter of tool invocations by tool name.
- `task.failure_rate` – ratio of failed tasks.
