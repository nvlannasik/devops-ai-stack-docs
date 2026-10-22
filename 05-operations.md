# Operations Guide

> **Audience:** on-call and platform engineers. Parent: *DevOps AI Agent Stack — Overview*.

## Health at a glance

| Check | How |
|---|---|
| Agent readiness | `GET /health` on port 3000 → `{"ok":true,"checks":{"mcp":"up","postgres":"up","redis":"up"}}`. It returns `503` when a configured dependency is down, so Kubernetes stops routing to that pod. |
| Running build | `kubectl -n devops-tools exec deploy/devops-ai-agent -- printenv GIT_SHA` |
| Tools discovered | Agent boot log: `Discovered 63 MCP tools` (55 read + 8 write) |
| Memory | Boot log: `Redis connected …` and `Incident memory: Postgres …` |

## The dashboard

The dashboard is read-only and served on port `3001` of the agent, behind one shared password. Expose it through the agent chart's optional `ingress` (route it to port `3001`) and set `dashboard.publicUrl`, so the postmortem links the agent posts in Slack open from a phone too. Without an Ingress, reach it with:

```bash
kubectl -n devops-tools port-forward deploy/devops-ai-agent 3001:3001
# open http://localhost:3001  (shared password from the DASHBOARD_PASSWORD Secret)
```

| Page | What it shows |
|---|---|
| `/` Overview | Incident counts and trends, recent incidents, remediation outcomes, model activity and **LLM cost per backend** |
| `/incidents`, `/incidents/<id>` | The filterable list. Each incident shows its RCA parsed into sections, change timeline, related alerts, remediations and verdicts, LLM usage and cost, and the harness trace. |
| `/postmortems` | Every incident that has a postmortem draft, newest first, with who asked for it, the backend and how many revisions |
| `/incidents/<id>/postmortem` (`.md`) | The latest postmortem draft, rendered or as a raw Markdown download |
| `/cluster` | The cluster inventory as a map: namespace → host → Service → workload → what manages it |
| `/memory` (Alert memory) | What the agent will recall for each alert + namespace, in the exact recall text it reads |
| `/memory/threads` (Conversation memory) | Per Slack thread: the message window, TTL, RCA flag and carried playbooks |
| `/harness`, `/harness/<gate>` | Every gate's activity across recorded runs, and LLM latency per backend |
| `/topology` | The agent stack's own map: LLM backends, MCP tools, Postgres, Redis and SQS |
| `/context`, `/prompt` | The skill registry, the token budget per backend, and the system prompt in use |
| `/bench` | Benchmark history: pass rates per scenario and axis |

## Logs and tracing an incident end to end

- Every log line of an investigation is prefixed with the **Slack thread id**, and the same id travels to the llm-worker as `traceId`.
- One grep therefore follows an incident across services:

```bash
kubectl -n devops-tools logs deploy/devops-ai-agent    | grep 1792547046.423669
kubectl -n devops-tools logs deploy/devops-llm-worker  | grep 1792547046.423669
```

- Useful markers: `Investigation started` / `Investigation complete in …ms (N LLM calls, M tool calls)`, `→ tool:` / `← tool:`, `[remediation] …`, `[slack] RCA card: …`, `[reconcile] …`.
- Delegated sub-investigations log as `<thread>/sub-1`, `<thread>/sub-2`.
- The full structured record of every run (each LLM turn, tool call and gate decision) is in Postgres `agent_events`, and is browsable from the incident page.

## Common tasks

| Task | How |
|---|---|
| Deploy new code | Push to `main` → wait for the GitHub Actions build → `kubectl -n devops-tools rollout restart deploy/<service>` → check `GIT_SHA` |
| Change chart values or versions | Commit to `gitops-devops-ai-manifest` (Flux applies it) |
| Add or edit a playbook | Edit `prompts/skills/*.md` in the agent repo; the tests validate every skill file |
| Add an LLM backend | Add `LLM_BACKEND_<N+1>_*` (indices must be contiguous) and name it in `LLM_ROUTE_LIGHT` or `LLM_ROUTE_HEAVY` |
| Price a backend | `LLM_BACKEND_<N>_PRICE_INPUT/_OUTPUT/_CACHE_READ` (USD per million tokens) |
| Allow remediation in a namespace | Add it to `ALLOWED_REMEDIATION_NAMESPACES` on the MCP server |
| Turn off remediation entirely | Set `MCP_ENABLE_WRITE_TOOLS=false` on the MCP server. The write tools are then not even registered. |
| Run the benchmark | Silence `namespace=~bench-.*` in Alertmanager first, or every injected fault also pages the production agent |

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Alert fired, nothing in Slack | Webhook rejected (`401`), or the alert was deduplicated in the last 12h | Agent log for `alert webhook received`. Check that the receiver's `credentials` equal `ALERT_WEBHOOK_TOKEN`. |
| Mention ignored, or an ephemeral "not allowed" message | The user is not in `SLACK_ALLOWED_USERS` | Add the user's Slack ID |
| Bot completely silent | Slack tokens or Socket Mode | Agent log for `[slack] unhandled error`; check `SLACK_APP_TOKEN` |
| Reply says the investigation exceeded its time budget | A slow LLM backend or a hung upstream | The footer names the backend. Check the worker log for that thread and the router's cool-off messages. |
| *Approve* does nothing or refuses | Card expired (15 min), the clicker is not an approver, or Interactivity is off | Card text; `SLACK_APPROVER_USERS`; Slack app Interactivity |
| No PR for a Flux workload | `GITOPS_REMEDIATION_ENABLED=false`, worker GitHub credentials, or drift | Worker log for the `requestId`. On drift the agent proposes `flux_reconcile` instead. |
| An RCA says "no logs" while logs exist | The LogQL label or the PromQL metric name does not match what the shipper or the app emits | The tool result's explained-empty note (`noLogLines`, `unknown_metric`) |
| A grounding warning under the RCA | The answer names a resource no tool returned | Treat the RCA as suspect; ask a follow-up in the thread |
| "I restarted before I could answer this" in a thread | A pod restart interrupted the work | Mention the agent again in the thread |
| Pod won't start | An invalid `LLM_BACKEND_*` set, a malformed skill file, or a failed migration | The pod's first log lines name the cause |
| Dashboard answers `503` | `DASHBOARD_PASSWORD` is not set | The Secret and the env var |

## Known limits

- **Pattern-based defences.** Masking and injection framing are pattern-based (see *Remediation and Safety Guardrails*).
- **Correlation is replica-local while in flight.** An investigation still in flight is visible to correlation only on the replica that runs it.
- **The RCA is a hypothesis.** Its quality depends on the LLM backend that answered; the footer always names that backend.
