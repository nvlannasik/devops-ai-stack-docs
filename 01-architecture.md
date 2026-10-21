# Architecture

> **Audience:** engineers. Parent: *DevOps AI Agent Stack — Overview*.

## Components

### devops-ai-agent (the orchestrator)
A Node.js 24 / TypeScript service. One process holds:
- **Slack integration** (Bolt, Socket Mode — outbound WebSocket, no public URL needed): mentions, ✅ reactions, approval buttons.
- **Alertmanager webhook** `POST /alert` (bearer-token protected). It answers `200` at once and investigates in the background.
- **Investigation loop**: builds the context, calls the LLM, executes tool calls in parallel, and runs the gates.
- **Remediation flow**: proposal → gate chain → dry-run → approval card → execution → verification poller.
- **Incident lifecycle**: stores RCAs, handles resolved alerts, runs a sweeper for resolves that never arrived, and handles `learn`, `postmortem` and status commands.
- **Dashboard**: a read-only second HTTP listener (port `3001`).
- **Readiness** `GET /health`: `503` when MCP, Postgres or Redis is unreachable.

### devops-mcp-server (the tools)
An MCP server over HTTP (`/mcp`, bearer token) or stdio. Tools are **auto-discovered** by the agent: a new tool needs no agent change.

| Domain | Count | Examples |
|---|---|---|
| Kubernetes | 37 | `k8s_cluster_health`, `k8s_cluster_inventory`, `k8s_change_timeline`, `k8s_correlate_pods`, `k8s_describe_pod`, `k8s_get_pod_logs` |
| Prometheus | 6 | `prometheus_query`, `prometheus_query_range`, rules, targets |
| Alertmanager | 1 | `alertmanager_get_alerts` — the only source of truth for "what is firing" |
| Loki | 6 | `loki_query`, `loki_query_range`, labels, streams |
| Tracing | 3 | `tracing_search`, `tracing_get_trace` (Tempo or Jaeger) |
| Capacity | 2 | `k8s_find_unused_resources`, `k8s_recommend_resources` |
| **Write** | 8 | Registered only with `MCP_ENABLE_WRITE_TOOLS=true`, and limited to `ALLOWED_REMEDIATION_NAMESPACES` |

An empty result is always **explained** (for example `noLogLines: true` with a verdict), never a bare `[]`. This keeps "the query was wrong" distinguishable from "nothing happened".

### llm-worker (the private-network bridge)
An SQS consumer that runs **inside the private network** and needs **outbound access to AWS SQS only**. It bridges two things:
1. **Agent → private LLM.** It speaks OpenAI `/v1/chat/completions`, Anthropic `/v1/messages`, or an agent-builder (Langflow) run endpoint. It translates content blocks natively and never stringifies them.
2. **Agent → GitHub Enterprise**, for the GitOps PR flow (`dry_run`, `open_pr`, `history`, `revert_pr`, `locate`). The worker holds the GitHub credentials (a PAT or a GitHub App); the agent holds none.

It drains gracefully on SIGTERM, sends poison messages to a DLQ, and processes requests concurrently (`SQS_MAX_CONCURRENCY`).

## Deployment topology (today)

![DevOps AI Agent Stack architecture](diagrams/architecture/architecture.png)

*Diagram made with archify. Interactive version: `diagrams/architecture/architecture.html`; vector: `diagrams/architecture/architecture.svg`.*

Everything is deployed by **Flux** from `gitops-devops-ai-manifest`. The three services ship as **one Helm release** (`devops-ai-stack` umbrella chart).

## Data stores

| Store | What it holds | Lifetime |
|---|---|---|
| **Postgres** | `incidents` (RCA, labels, change timeline, parent/child correlation), `incident_feedback` (human-confirmed causes), `remediations` (proposal, approval, result, backup), `remediation_checks` (verification due dates and verdicts), `llm_usage` (tokens per call and backend), `agent_events` (the full trace of every run), `postmortems` | Durable. The schema is applied by versioned migrations at pod startup (advisory-locked). |
| **Redis** | `conv:*` conversation history per Slack thread, `rca:*` "RCA already sent" flag, `skills:*` the playbooks a thread carries, `dedup:*` alert fingerprints | 24h conversation TTL; 12h dedup claims |

Conversation memory (Redis) and incident memory (Postgres) are deliberately separate: one is a short-lived cache, the other is the agent's long-term memory.

## Cross-service contracts

These are the interfaces where a change on one side breaks the other. They must change on both sides together.

| Contract | Transport | Shape (summary) |
|---|---|---|
| agent ↔ mcp-server | MCP over HTTP, `Authorization: Bearer <MCP_AUTH_TOKEN>` | Standard MCP `listTools` / `callTool`. Write tools carry a `[WRITE]` description prefix. |
| agent ↔ llm-worker (LLM) | SQS FIFO: `llm-request.fifo` → shared `llm-response.fifo` | `{ requestId, messages, tools, systemPrompt, traceId? }` → `{ requestId, response \| error }`. Routed by `requestId`. |
| agent ↔ llm-worker (GitOps) | SQS FIFO: `gitops-request.fifo` → the same shared response queue | `{ requestId, op: dry_run \| open_pr \| history \| revert_pr \| locate, … }`. A refusal can carry `drift` (the repo and the cluster disagree). |
| agent ↔ mcp-server bridge (multi-cluster) | SQS FIFO: `devops-mcp-<cluster>-request.fifo` → `devops-mcp-response.fifo` | `{ requestId, op: list_tools \| call_tool, tool?, args?, deadline, traceId? }`. A request is never retried. A request past its deadline is answered `expired` and never run. A write message is deleted before it runs (at-most-once). |

`traceId` is the Slack thread id. One `grep <thread-id>` therefore joins the agent log, the llm-worker log and the Slack thread.

## LLM routing

With `LLM_PROVIDER=router` the agent holds several backends and picks one per call:
- **light route** — plain Slack mentions and formatting backstops;
- **heavy route** — alerts, explicit investigation requests, remediation proposals.

Failover is **up-only** (light → heavy, never the reverse), because a weak model fails by answering confidently, not by throwing an error. A backend that fails repeatedly is skipped for a cool-off. Only backends of kind `private-llm` use the SQS path; `claude` and `openai-compatible` backends are called directly.

The dev cluster currently runs four backends: two OpenAI-compatible APIs and two private LLMs over SQS.

## Multi-cluster (in progress)

The target topology is a **central cluster** that hosts the observability stack and the agent, and **environment clusters** that are reachable only outbound.

Each environment cluster runs:
- a copy of `devops-mcp-server` in **SQS bridge mode** (`TRANSPORT=sqs`);
- metrics and log shippers that write to the central Prometheus and Loki with a `cluster` label.

Part 1 (the bridge) is live. Part 2 routes every Kubernetes tool call of an investigation to that investigation's cluster, while observability queries go to the central stack filtered by the `cluster` label. Part 2 is in design.
