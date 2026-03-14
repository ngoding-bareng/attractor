# Custom Invention Spec for Forked Attractor

This document records extensions, inventions, and discrepancies between the original attractor spec and the xenith-attractor implementation.

## 1. Inventions (New Concepts Added to Original Spec)

### A. HTTP Server Mode

A stateless HTTP API wrapping the attractor runner for long-lived, polling-friendly execution:

```
POST /runs
  Input: { dot: string, context: map[string]any }
  Output: { id: string }
  Effect: Start async pipeline execution, return run ID

GET /runs/:id
  Output: { status: string, completed_nodes: []string, elapsed: duration }
  Status: "pending" | "running" | "succeeded" | "failed"

GET /runs/:id/events
  Content-Type: text/event-stream
  Output: SSE stream forwarding OnEvent callbacks in real-time

GET /runs/:id/graph
  Output: { dot: string, node_status: map[string]string }
  Effect: Return original DOT + per-node status overlay for browser rendering

POST /runs/:id/feedback
  Input: { message: string } or { steering: map[string]any }
  Effect: Inject message/steering into running agent session
```

Use cases:
- Long-running pipelines (hours) without blocking HTTP connection
- Browser-based progress tracking
- Checkpoint/recovery across restarts

---

### B. Web UI / Real-time Progress Dashboard

A browser-based pipeline visualization consuming the HTTP server API:

Features:
- **Graph rendering**: Render DOT graph using vis.js or d3-graphviz
- **Node status overlays**: Visual indicators (pending → running → success/fail) on each node
- **Live event stream**: SSE connection shows logs, tool calls, and node transitions in real-time
- **Clickable inspection**: Click a node to see agent tool call history, reasoning, and output
- **Feedback injection**: Text input field to steer running agent mid-pipeline

Example flow:
1. User submits DOT + context via UI form
2. UI polls `/runs/:id` or streams `/runs/:id/events`
3. Graph nodes highlight as they progress
4. User clicks a node → sidebar shows agent session transcript
5. User types feedback → POST to `/runs/:id/feedback` → agent incorporates it

---

### C. AgentLoopBackend as CodergenBackend

Integration of the `agentloop` multi-turn agent as a first-class pipeline node type:

**Library Mode (not subprocess)**
- `AgentLoopBackend` runs as an in-process library, not a spawned subprocess
- Each `codergen` node executes a full multi-turn agent session:
  - Read files via tool
  - Write code/docs via tool
  - Execute shell commands
  - Search codebase via grep
  - Fetch web content
  - Delegate to subagents

**Per-Node Filesystem Scope**
- Node attribute `working_dir` sets the agent's filesystem scope
- Falls back to `ctx.GetString("working_dir", ".")` if absent
- Enables **one DOT file to drive agents across multiple repos**

**Event Forwarding**
- `BridgeEmitter` converts `agentloop.OnEvent` callbacks to `attractor.OnEvent`
- Streaming logs and tool calls flow transparently into the DAG's event stream
- Parent pipeline can subscribe to agent internals without special plumbing

Example node:
```dot
analyze_repo [
  type=codergen
  prompt="Identify the main entry point and summarize architecture"
  working_dir="/repos/backend"
]
```

---

### D. Self-Modifying Pipelines (DOT-as-Artifact)

A pipeline can generate and execute another pipeline:

**Pattern**
1. Planner node writes a new DOT file to disk using `write_file` tool
2. Downstream `stack.manager_loop` node loads and executes that DOT
3. Creates a feedback loop: agent plans its own pipeline → executor runs it

Example:
```dot
plan_migrations [
  type=codergen
  prompt="Analyze SQL migrations, plan a refactoring strategy, write plan.dot"
  working_dir="/repos/database"
]

execute_plan [
  type=stack.manager_loop
  dot_file="plan.dot"
]

plan_migrations -> execute_plan
```

The `plan_migrations` agent:
1. Reads migration files
2. Calls `write_file("plan.dot", "digraph { ... }")` to output a DAG
3. Completes

The `execute_plan` node:
1. Loads the generated `plan.dot`
2. Runs it as a sub-pipeline
3. Reports success/failure upstream

---

### E. Per-Node Working Directory (`working_dir` attribute)

A **new** node attribute (not in original spec) enabling multi-repo pipelines:

**Specification**
- Node attribute: `working_dir="/path/to/repo"`
- Type: optional string
- Default: `"."` (current directory)
- Effect: `AgentLoopBackend` calls `LocalExecEnv.SetWorkingDir(working_dir)` before running
- Scope: Only affects that node; siblings may have different `working_dir` values

**Use cases**
- Single pipeline orchestrating work across multiple Git repositories
- Monorepo scenarios: separate `working_dir` per service
- Multi-tenant demos: each tenant's repo gets its own agent

Example:
```dot
digest: graphviz=true

audit_service_a [
  type=codergen
  prompt="Audit code quality"
  working_dir="/repos/service-a"
]

audit_service_b [
  type=codergen
  prompt="Audit code quality"
  working_dir="/repos/service-b"
]

summarize [
  type=codergen
  prompt="Combine results from both audits"
  working_dir="."
]

audit_service_a -> summarize
audit_service_b -> summarize
```

---

### F. Optional Worktree Isolation per AgentLoopBackend Invocation

Safe parallel execution by isolating agents in Git worktrees:

**Pattern**
1. Before calling `agentloop.Run()`, create a temporary worktree:
   ```go
   worktreeDir := "/tmp/wt-" + uuid.New().String()
   exec.Command("git", "worktree", "add", worktreeDir, "HEAD").Run()
   ```
2. Pass `worktreeDir` as `working_dir` to the agent
3. Agent makes changes in isolation
4. After session completes:
   - For PR workflow: keep worktree, let caller manage git push
   - For disposable runs: clean up with `git worktree remove`

**Benefits**
- Multiple agents can safely run on the same repo simultaneously
- Each agent's changes are isolated; no merge conflicts mid-session
- Parent repo remains clean; no uncommitted state leakage

---

### G. Daily Automation Pattern

Scheduled pipelines with checkpoint/recovery between runs:

**Architecture**
- External scheduler (cron, K8s CronJob, Temporal, etc.) invokes attractor runner
- Runner passed `--logs-root /logs/$(date +%Y%m%d)` flag
- Logs and checkpoint files are dated per run
- If previous run failed, checkpoint is replayed before starting

Example cron job:
```bash
0 9 * * * /usr/local/bin/attractor run \
  --dot /home/ops/daily-tasks.dot \
  --context-file /home/ops/context.json \
  --logs-root /logs/$(date +\%Y\%m\%d)
```

**No HTTP Server Required**
- Simple scheduled pipelines don't need HTTP endpoints
- Logs are written to disk for later inspection
- Runner exits when pipeline completes

**Checkpoint/Resume**
- Before each node, runner checks `--logs-root` for prior state
- If node already completed, skip and load cached result
- Enables crash recovery: restart pipeline, failed nodes retry from checkpoint

---

## 2. Discrepancies (Original Spec vs konco-attractor)

| Feature | Original Spec | konco-attractor | Gap | Severity |
|---------|---|---|---|---|
| **HTTP Server Mode** | REST endpoints: POST /runs, GET /runs/:id, GET /runs/:id/events, GET /runs/:id/graph, POST /runs/:id/feedback | Missing | Not implemented; requires separate server wrapper | Medium |
| **Fidelity Modes** | Four modes: `full`, `truncate`, `compact`, `summary` for prompt injection | Missing | Only basic token truncation in agentloop; no structured fidelity API | Low |
| **Thread ID Resolution** | Per-node thread IDs; LLM session reuse across nodes | Missing | Each node starts a fresh agentloop session; no cross-node LLM state | Medium |
| **Artifact Store** | File-backed structured API: `GetArtifact()`, `SetArtifact()` with schema | Partial | Writes `response.md` per stage; no structured schema/query API | Low |
| **Preamble Generation Transform** | Built-in graph-level preamble; injected into all node prompts | Missing | No preamble transform; each node prompt is independent | Low |
| **Custom Transform Registration** | Public interface for custom AST transforms | Partial | `ApplyBuiltInTransforms()` exists; no registration API for user transforms | Low |
| **ParallelHandler Concurrency** | Parallel branches execute as concurrent goroutines | Sequential | `for` loop iterates branches one-by-one; no fan-out | High |
| **`followup_queue` in Session** | Queue for user messages after current input completes | Missing | `steering_queue` exists for steering input; no followup queue semantics | Low |
| **`AWAITING_INPUT` State** | Model pauses mid-session to ask user a question | Missing | Session states: IDLE, PROCESSING, CLOSED; no pause-for-input | Medium |
| **Variable Expansion** | Full support: `$goal`, `$context`, custom variables | Partial | `$goal` and `$context` work; no extensible variable API | Low |

---

## 3. Planned Extensions (Not Yet Spec'd)

### A. konco-attractor-server
Standalone HTTP server wrapping the existing attractor runner:
- Listens on `:8080` (configurable)
- Forwards `OnEvent` callbacks to SSE clients
- Manages run storage (in-memory or Redis)
- Ready for deployment behind a load balancer

### B. konco-attractor-ui
Browser-based dashboard:
- React or HTMX frontend
- Consumes `konco-attractor-server` API
- Renders live DOT graph with vis.js or d3-graphviz
- Clickable nodes show agent session transcripts
- Feedback injection form for steering

### C. Parallel Branch Concurrency Fix
Replace sequential `for` loop in `ParallelHandler.Run()` with goroutine fan-out:
- Branches execute concurrently
- Results synchronized before downstream nodes
- Reduces wall-clock time for wide parallelism

### D. Formalize `working_dir` in DSL Schema
Add `working_dir` to the official DOT spec:
- Document attribute name and type
- Define semantics (affects `LocalExecEnv` scope)
- Examples in DSL guide

---

## Verification Checklist

- [x] 7 inventions documented (A–G) with examples
- [x] Discrepancy table complete with severity ratings
- [x] Planned extensions listed (4 items)
- [x] No original spec files modified
- [x] All DOT examples valid graphviz syntax
