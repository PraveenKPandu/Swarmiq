# Data Flow

This document describes how data moves through the Swarmiq platform, covering the lifecycle of a swarm run, inter-agent communication patterns, and key sequence diagrams.

---

## 1. Swarm Run Lifecycle

A **swarm run** begins when a client submits a goal and ends when the final answer is delivered. The stages are:

```
Client Submits Goal
        │
        ▼
┌──────────────────┐
│  API Gateway     │  Authenticates request, validates payload
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Swarm Orchestr.  │  Creates a Swarm record, transitions to PLANNING
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Planner       │  Decomposes goal → task graph (DAG)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Task Manager    │  Persists tasks; marks root tasks as READY
└────────┬─────────┘
         │
         ▼  (repeated per task, potentially in parallel)
┌──────────────────┐
│   Agent Pool     │  Pulls READY task, assigns to an Agent Worker
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Agent Runner    │  Executes ReAct loop; calls tools as needed
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Task Manager    │  Receives result; marks task COMPLETE;
│                  │  unblocks dependent tasks
└────────┬─────────┘
         │
         ▼  (when all tasks are COMPLETE)
┌──────────────────┐
│ Swarm Orchestr.  │  Assembles final answer; transitions to COMPLETE
└────────┬─────────┘
         │
         ▼
  Client Receives Result
```

---

## 2. Sequence Diagrams

### 2.1 Submit Goal and Receive Result (Synchronous REST)

```
Client          API GW        Orchestrator     Planner      Task Mgr    Agent
  │                │                │              │             │          │
  │──POST /swarms──▶               │              │             │          │
  │                │─authenticate──▶              │             │          │
  │                │──create swarm──▶             │             │          │
  │                │                │──plan goal──▶             │          │
  │                │                │◀────task graph────────────│          │
  │                │                │──persist tasks────────────▶          │
  │                │                │                           │          │
  │◀──202 swarm_id─│                │                           │          │
  │                │                │                           │──READY──▶│
  │                │                │                           │          │
  │                │                │                           │◀─result──│
  │                │                │◀──────swarm.complete──────│          │
  │                │                │                           │          │
  │──GET /swarms/id▶               │              │             │          │
  │◀──200 result───│◀──────────────▶              │             │          │
```

### 2.2 Real-Time Streaming (WebSocket)

```
Client          API GW        Orchestrator                   Agent
  │                │                │                           │
  │──WS Connect────▶               │                           │
  │──send goal─────▶               │                           │
  │                │──create swarm──▶                          │
  │                │                │──dispatch task────────────▶
  │◀─task.created──│◀───────────────│                          │
  │                │                │                          │──thinking
  │◀─agent.thinking│◀───────────────────────────────────────────│
  │                │                │                          │──tool_call
  │◀─agent.tool────│◀───────────────────────────────────────────│
  │                │                │                          │──result
  │◀─agent.result──│◀───────────────────────────────────────────│
  │◀─swarm.complete│◀───────────────│                          │
```

### 2.3 Agent ReAct Loop (Internal)

```
Agent Runner       LLM API        Tool Executor     Memory Store
     │                │                 │                │
     │──load context──────────────────────────────────────▶
     │◀─context snippets──────────────────────────────────│
     │                │                 │                │
     │──prompt+tools──▶                │                │
     │◀──think+act────│                │                │
     │                │                │                │
     │──[tool_name, args]──────────────▶                │
     │◀──[tool_result]─────────────────│                │
     │                │                │                │
     │──inject result─▶                │                │
     │◀──final answer─│                │                │
     │                │                │                │
     │──store finding─────────────────────────────────────▶
     │──return result to Task Manager  │                │
```

### 2.4 Multi-Agent Collaboration (Parallel Execution)

```
Orchestrator    Task Mgr    Agent A (Researcher)    Agent B (Analyst)    Agent C (Writer)
     │              │               │                      │                   │
     │──plan────────▶              │                      │                   │
     │              │──READY(T1)───▶                      │                   │
     │              │──READY(T2)──────────────────────────▶                   │
     │              │               │ [T1: research]       │ [T2: data analysis│
     │              │               │ (parallel)           │ (parallel)        │
     │              │◀──result T1───│                      │                   │
     │              │◀──result T2──────────────────────────│                   │
     │              │──READY(T3, deps: T1+T2)──────────────────────────────────▶
     │              │                                                          │ [T3: write]
     │              │◀──result T3──────────────────────────────────────────────│
     │◀─complete────│               │                      │                   │
```

### 2.5 Human-in-the-Loop Feedback

```
Client          Orchestrator        Agent              Human Feedback Handler
  │                  │                │                        │
  │                  │──dispatch task─▶                       │
  │                  │                │──needs approval?       │
  │                  │                │────────────────────────▶
  │◀─human.feedback_required──────────────────────────────────│
  │                  │                │                        │
  │──send feedback───────────────────────────────────────────▶│
  │                  │                │◀─────feedback injected─│
  │                  │                │──continue reasoning    │
  │                  │                │──result──────────────────
  │◀─agent.result────│◀───────────────│                        │
```

---

## 3. Inter-Agent Communication Patterns

### 3.1 Orchestrator-to-Agent (Hierarchical)

The default pattern. The Orchestrator is the sole dispatcher; agents never communicate directly. This maximises observability and simplifies debugging.

```
Orchestrator
    ├──▶ Agent A
    ├──▶ Agent B
    └──▶ Agent C
```

### 3.2 Agent Spawning Sub-Agents (Recursive Delegation)

An agent may spawn sub-agents via the `spawn_sub_agent` tool when it determines its task requires specialised help. The spawned sub-swarm is managed by a nested Orchestrator instance.

```
Orchestrator
    └──▶ Agent (Lead)
              └──▶ Sub-Orchestrator
                        ├──▶ Sub-Agent X
                        └──▶ Sub-Agent Y
```

### 3.3 Peer Agent Query (Lateral Communication)

An agent can query another running agent for a piece of information using the `query_agent` tool. This is mediated through the Message Bus so it remains observable and auditable.

```
Agent A ──[query_agent(agent_b, question)]──▶ Message Bus ──▶ Agent B
Agent A ◀─[answer]────────────────────────── Message Bus ◀── Agent B
```

---

## 4. Memory Read/Write Patterns

### 4.1 Write on Completion (Default)

After every task, the Agent Runner extracts key facts and stores them in Episodic Memory. This makes findings available to downstream agents.

```
Agent Runner  ──result──▶  Task Manager
              ──extract──▶ Memory Extractor  ──embed──▶ Vector DB
```

### 4.2 Context Injection on Task Start

Before the Agent Runner constructs its LLM prompt, it queries Memory Store for relevant context using the task goal as the search query. The top-k results are injected into the system prompt.

```
Agent Runner  ──goal query──▶  Memory Store  ──top-k snippets──▶  Prompt Builder
```

### 4.3 Cross-Swarm Knowledge Retention

Semantic Memory persists across swarm runs. When a new swarm starts, the Planner queries Semantic Memory for relevant prior knowledge, potentially reducing redundant agent work.

---

## 5. Error Handling and Recovery

### 5.1 Agent Failure

```
Agent Runner ──error──▶ Task Manager
                              │
               ┌──────────────┼───────────────────┐
               ▼              ▼                   ▼
        retryable?        timeout?         hard failure?
           │                 │                   │
      re-queue task    re-queue with         mark FAILED
      (new agent)      shorter budget        notify Orch.
```

### 5.2 Cascading Failure Prevention

If a critical-path task fails beyond the retry limit, the Swarm Orchestrator:
1. Cancels all tasks that depend on the failed task.
2. Attempts an alternative plan path if one was generated by the Planner.
3. Marks the swarm as `FAILED` and returns a structured error to the client if no alternative exists.

### 5.3 Timeout Handling

Each task has a `deadline` field. The Task Manager runs a background sweep every 30 seconds and marks timed-out tasks as `FAILED`, triggering the failure handling flow above.

---

## 6. Data Schemas

### Task Object

```json
{
  "id": "task_01j2k...",
  "swarm_id": "swarm_01j2k...",
  "parent_task_id": null,
  "goal": "Research the latest advancements in quantum computing",
  "agent_type": "researcher",
  "status": "COMPLETE",
  "context": {
    "memory_snippets": ["..."],
    "prior_results": []
  },
  "constraints": {
    "max_tokens": 4096,
    "timeout_seconds": 120,
    "allowed_tools": ["web_search", "url_fetch"]
  },
  "result": {
    "content": "...",
    "token_usage": { "prompt": 812, "completion": 432 },
    "tool_calls": [{ "tool": "web_search", "args": { "query": "..." }, "result": "..." }]
  },
  "created_at": "2025-01-01T12:00:00Z",
  "completed_at": "2025-01-01T12:01:45Z"
}
```

### Swarm Object

```json
{
  "id": "swarm_01j2k...",
  "goal": "Write a comprehensive report on quantum computing trends",
  "status": "COMPLETE",
  "plan": {
    "strategy": "parallel",
    "tasks": ["task_01j2k...", "task_01j2l...", "task_01j2m..."]
  },
  "result": {
    "content": "...",
    "total_token_usage": { "prompt": 4210, "completion": 1843 },
    "duration_seconds": 47.3
  },
  "created_at": "2025-01-01T12:00:00Z",
  "completed_at": "2025-01-01T12:00:47Z"
}
```
