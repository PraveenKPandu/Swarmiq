# Architecture Decision Records (ADRs)

Architecture Decision Records capture the context, decision, and consequences of significant architectural choices made in the Swarmiq project. They are immutable: once recorded, a decision is only superseded by a new ADR, never edited.

---

## ADR Index

| ID | Title | Status |
|---|---|---|
| [ADR-001](#adr-001) | Python as the primary implementation language | Accepted |
| [ADR-002](#adr-002) | Event-driven, asynchronous agent communication | Accepted |
| [ADR-003](#adr-003) | Pluggable LLM backend via a unified adapter | Accepted |
| [ADR-004](#adr-004) | ReAct (Reason + Act) as the default agent loop | Accepted |
| [ADR-005](#adr-005) | Pydantic v2 for data modelling and validation | Accepted |
| [ADR-006](#adr-006) | FastAPI for the HTTP/WebSocket API layer | Accepted |
| [ADR-007](#adr-007) | Vector database for semantic agent memory | Accepted |
| [ADR-008](#adr-008) | Task graph (DAG) over linear task chains | Accepted |
| [ADR-009](#adr-009) | OpenTelemetry for observability | Accepted |
| [ADR-010](#adr-010) | Apache 2.0 open-source licence | Accepted |

---

## ADR-001

**Title:** Python as the primary implementation language

**Status:** Accepted

**Date:** 2025-01

### Context

The platform needs to integrate deeply with AI/ML libraries, LLM client SDKs, and vector databases. Several implementation language options were considered.

### Decision

Python 3.11+ is the primary language for the Swarmiq platform core.

### Reasoning

- The AI/ML ecosystem (LangChain, LlamaIndex, OpenAI SDK, Anthropic SDK, Qdrant client, etc.) is predominantly Python-first.
- Python's `asyncio` / `anyio` libraries provide first-class async concurrency without the complexity of compiled languages.
- Python has mature packaging tools (`uv`, `pyproject.toml`) that support reproducible builds.
- The target contributor audience (AI engineers, ML practitioners) is most proficient in Python.

### Consequences

- Agent implementations in other languages must be wrapped as subprocess tools or HTTP microservices.
- Performance-critical hot paths (token streaming, embedding computation) may need profiling and optimisation, but are not expected to be bottlenecks given I/O-bound workloads.

---

## ADR-002

**Title:** Event-driven, asynchronous agent communication

**Status:** Accepted

**Date:** 2025-01

### Context

Agents need to communicate progress, results, and errors to the orchestrator and the client. The communication model could be synchronous (direct function calls / RPC) or asynchronous (message bus).

### Decision

All inter-component communication (except configuration reads) is asynchronous and event-driven, routed through a Message Bus.

### Reasoning

- **Decoupling**: components do not need to know about each other's interfaces.
- **Scalability**: multiple consumers can subscribe to the same topic independently.
- **Observability**: all messages pass through a single observable channel.
- **Resilience**: producers do not block waiting for consumers; consumers can catch up after a restart.

### Consequences

- Debugging becomes slightly harder because the call stack is no longer a simple linear trace. This is mitigated by distributed tracing via OpenTelemetry.
- Local development mode uses an in-process event emitter to avoid infrastructure dependencies.

---

## ADR-003

**Title:** Pluggable LLM backend via a unified adapter

**Status:** Accepted

**Date:** 2025-01

### Context

The LLM landscape is rapidly evolving. Locking into a single provider (e.g., OpenAI) would limit flexibility and create commercial risk.

### Decision

All LLM calls are made through a `LLMAdapter` abstraction that supports multiple backends (OpenAI, Anthropic, Google Gemini, Ollama/local models) via a common interface.

### Reasoning

- Organisations often require the ability to switch providers to manage cost, privacy, or capability trade-offs.
- Local/open-source models (via Ollama) are important for on-premises deployments and privacy-sensitive use cases.
- A single adapter interface simplifies testing (mock implementations, deterministic fixtures).

### Consequences

- Features that are provider-specific (e.g., OpenAI's structured outputs) must be exposed as optional capabilities in the adapter interface.
- Maintaining multiple adapter implementations requires ongoing engineering effort as provider APIs evolve.

---

## ADR-004

**Title:** ReAct (Reason + Act) as the default agent loop

**Status:** Accepted

**Date:** 2025-01

### Context

Several agent reasoning patterns exist: plain chain-of-thought, ReAct, Plan-and-Execute, Reflexion, and Tree-of-Thoughts. The platform needs a sensible default.

### Decision

The default agent execution loop implements **ReAct** (Yao et al., 2022): alternating Thought → Action → Observation steps until a final answer is produced.

### Reasoning

- ReAct is well-understood, widely used, and has strong empirical benchmarks.
- It naturally supports tool use (the Action step) and introspection (the Thought step).
- The loop is simple to implement, inspect, and debug.
- Advanced loops (Plan-and-Execute, Reflexion) can be layered on top or offered as alternative agent types.

### Consequences

- Agents may enter infinite loops if the LLM does not converge. A hard cap on loop iterations (default: 15) is enforced.
- The sequential nature of ReAct means that an agent cannot exploit intra-turn parallelism; parallel work requires the Planner to decompose into multiple agents.

---

## ADR-005

**Title:** Pydantic v2 for data modelling and validation

**Status:** Accepted

**Date:** 2025-01

### Context

The platform exchanges structured data (tasks, results, tool schemas, events) between many components. A data validation library is needed.

### Decision

All data models are defined using Pydantic v2.

### Reasoning

- Native integration with FastAPI (request/response models, OpenAPI schema generation).
- Pydantic v2 is significantly faster than v1 due to its Rust-backed validation core.
- Rich validation features: field validators, model validators, discriminated unions.
- JSON serialisation is built-in and configurable.

### Consequences

- Pydantic v2 has a different API from v1; contributors familiar with v1 must account for breaking changes.
- All tool input/output schemas are generated from Pydantic models, ensuring consistency between documentation and runtime validation.

---

## ADR-006

**Title:** FastAPI for the HTTP/WebSocket API layer

**Status:** Accepted

**Date:** 2025-01

### Context

A production-grade HTTP API framework is needed for the Client Layer.

### Decision

The REST and WebSocket APIs are built with FastAPI.

### Reasoning

- First-class async support matches the platform's async core.
- Automatic OpenAPI 3.1 spec generation reduces documentation overhead.
- Native Pydantic integration (ADR-005) means models are shared between API and business logic layers.
- Large community, extensive middleware ecosystem (CORS, rate limiting, auth).

### Consequences

- FastAPI is Python-only; this is acceptable given ADR-001.
- ASGI deployment (Uvicorn / Gunicorn + Uvicorn workers) is required; standard WSGI servers are not compatible.

---

## ADR-007

**Title:** Vector database for semantic agent memory

**Status:** Accepted

**Date:** 2025-01

### Context

Agents need to retrieve relevant prior knowledge using semantic similarity, not just exact key-value lookup.

### Decision

The Episodic and Semantic memory tiers use a **vector database** (Qdrant by default, with adapters for Weaviate, Pinecone, pgvector).

### Reasoning

- Semantic similarity search (nearest-neighbour over embeddings) is the appropriate retrieval mechanism for unstructured agent findings.
- Vector databases provide scalable approximate nearest-neighbour (ANN) search with filtering.
- Qdrant is chosen as the default because it is open-source, self-hostable via Docker, and has a rich Python client.

### Consequences

- Embeddings must be generated for every stored memory chunk; this has a latency and cost overhead.
- The choice of embedding model affects retrieval quality; the platform ships with a default but allows configuration.

---

## ADR-008

**Title:** Task graph (DAG) over linear task chains

**Status:** Accepted

**Date:** 2025-01

### Context

The Planner must represent dependencies between subtasks. Two structural options were considered: a linear chain (tasks run sequentially, each consuming the previous output) or a directed acyclic graph (tasks can run in parallel when they have no dependencies).

### Decision

Tasks are organised as a **Directed Acyclic Graph (DAG)**. Independent tasks are executed in parallel; dependent tasks wait for their prerequisites.

### Reasoning

- Many real-world goals decompose into independent subtasks (e.g., research topic A and topic B simultaneously, then synthesise).
- Parallel execution reduces end-to-end latency significantly.
- A DAG is a strict superset of a linear chain; sequential workflows are expressible as a degenerate DAG.

### Consequences

- The Task Manager must implement topological sorting and dependency tracking.
- Visualising task plans as graphs is more complex than displaying a list, but provides better insight into execution progress.

---

## ADR-009

**Title:** OpenTelemetry for observability

**Status:** Accepted

**Date:** 2025-01

### Context

The platform needs structured logging, distributed tracing, and metrics to support debugging and performance monitoring.

### Decision

All observability signals are emitted using the **OpenTelemetry** SDK, exported via the OTLP protocol.

### Reasoning

- OpenTelemetry is the CNCF-standard, vendor-neutral observability framework; signals can be sent to any compatible backend (Jaeger, Tempo, Prometheus, Datadog, Honeycomb, etc.).
- A single SDK covers all three signal types (logs, traces, metrics), reducing the number of dependencies.
- Distributed tracing is critical for debugging multi-agent runs where a single user request spawns dozens of LLM calls across multiple processes.

### Consequences

- A compatible OTLP receiver must be deployed in production (minimal overhead; Jaeger or OpenTelemetry Collector suffices).
- Developer environments default to stdout export to avoid requiring external services.

---

## ADR-010

**Title:** Apache 2.0 open-source licence

**Status:** Accepted

**Date:** 2025-01

### Context

The project is open-source. A licence must be chosen that balances openness with legal protection.

### Decision

Swarmiq is licensed under the **Apache License 2.0**.

### Reasoning

- Apache 2.0 is permissive: users can use, modify, and distribute the software commercially.
- Includes an explicit patent grant, providing contributors and users with IP protection.
- Compatible with most open-source AI/ML dependencies.
- Well-understood by enterprises, reducing adoption friction.

### Consequences

- Derived works must retain the original licence notice and NOTICE file.
- Copyleft (GPL-style) restrictions do not apply; third parties can incorporate Swarmiq into proprietary products.
