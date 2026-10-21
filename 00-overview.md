# DevOps AI Agent Stack — Overview

> **Audience:** everyone — engineers, on-call, and stakeholders.
> **Status:** running in the `dev` cluster (namespace `devops-tools`). Multi-cluster support is in progress (see *Roadmap*).

## What it is

The DevOps AI Agent Stack is an AI-assisted incident responder that lives in Slack. When an Alertmanager alert fires — or when an engineer asks a question — it investigates the Kubernetes cluster and its observability data the way an on-call engineer would, and posts a structured **Root Cause Analysis (RCA)** in the alert's Slack thread within minutes.

It can also **propose one fix** for the problem it found. Nothing is ever changed without a human clicking *Approve*, and every change is verified afterwards.

## Why we built it

| Problem | What the stack does about it |
|---|---|
| The first 15–30 minutes of an incident go to gathering the same evidence every time (pods, events, logs, metrics, recent changes). | The agent gathers that evidence automatically and in parallel, and starts the thread with an RCA and the evidence behind it. |
| Knowledge from past incidents lives in people's heads. | Every RCA is stored. Engineers can teach the agent the real root cause with `@agent learn`, and the next recurrence starts from it. |
| Fixes applied by hand under pressure bypass review and are not audited. | Fixes go through a whitelist, a gate chain, a mandatory dry-run, a human approval, and an audit trail. For GitOps-managed workloads the fix becomes a pull request. |
| New team members need a long time to learn what runs where. | `@agent explain this cluster` produces an onboarding tour built from the live cluster. |

## What it does today

- **Alert → RCA in Slack.** Root cause, evidence (with the tool that produced each fact), ruled-out hypotheses, recommended actions, impact and a confidence level. Low confidence automatically pages the on-call users.
- **Conversational follow-ups** in the thread ("show me the logs", "when did this start?").
- **Guarded remediation.** One whitelisted fix per incident, approval-gated, verified about 5 minutes later. GitOps pull requests are used for Flux-managed workloads.
- **Incident memory.** It recalls human-confirmed causes, past RCAs, related incidents and past fixes with their outcomes.
- **Change timeline.** It shows what changed in the namespace during the last 24 hours (rollouts, Helm upgrades, ConfigMaps, Git commits).
- **Cross-alert correlation.** A neighbouring alert joins the open incident's thread instead of opening a new one.
- **Cluster tour** for onboarding, and **postmortem drafts** on request.
- **Multi-LLM.** It can use a hosted model (Claude, an OpenAI-compatible API) or a private LLM inside the private network, with automatic failover.
- **Read-only dashboard.** Incidents, remediation outcomes, LLM cost, gate activity, the cluster map and the agent's memory.

## How it works (one picture)

```
 Alertmanager ──webhook──┐                       ┌──► Slack thread: RCA card, follow-ups,
 Slack @mention ─────────┤                       │    approval card, verdict
                         ▼                       │
                 ┌───────────────┐   MCP    ┌──────────────────┐
                 │ devops-ai-    │─────────►│ devops-mcp-server│──► Kubernetes API
                 │ agent         │◄─────────│ (55 read + 8     │──► Prometheus / Loki
                 │ (Slack bot +  │          │  write tools)    │──► Alertmanager / Tracing
                 │  agentic loop)│          └──────────────────┘
                 └──────┬────────┘
          SQS (FIFO)    │   Postgres (incident memory) · Redis (conversation memory)
                        ▼
                 ┌───────────────┐
                 │ llm-worker    │──► private LLM (inside the private network)
                 │ (private net) │──► GitHub Enterprise (GitOps PRs)
                 └───────────────┘
```

1. An alert (or a mention) arrives. The agent deduplicates it, correlates it with open incidents, recalls past incidents and builds the change timeline.
2. An LLM drives an **investigation loop**: it calls read-only tools on the MCP server (Kubernetes, Prometheus, Loki, Alertmanager, tracing), reads the results, and calls more until it can explain the incident.
3. Deterministic **gates** check the answer before it leaves. For example, an alert answered without evidence gets another round, and a claim with no supporting log is capped at Medium confidence.
4. The RCA is posted to Slack and stored in Postgres.
5. If a fix is warranted, the agent proposes one. A human approves it, the change runs, and the agent checks later whether it worked.

## Safety model in one paragraph

The investigating model **never has write access**: write tools are filtered out of the loop twice. A change can only happen through a separate proposal that must:
- be on a whitelist of reversible actions;
- pass a chain of deterministic checks;
- survive a server-side dry-run;
- target an allowed namespace;
- be **approved by a named human in Slack**.

On GitOps-managed workloads the change becomes a pull request instead of a live patch. Before any request leaves for an LLM, credential-like identifiers (IPs, hostnames, ARNs, emails) are masked. Text returned by the cluster (logs, events) is framed as data so it cannot instruct the agent.

## Components at a glance

| Component | Repository | Role |
|---|---|---|
| **devops-ai-agent** | `devops-ai-agent` | Slack bot, Alertmanager webhook, investigation loop, remediation flow, dashboard. The orchestrator. |
| **devops-mcp-server** | `devops-mcp-server` | MCP tool server: 55 read-only tools (Kubernetes, Prometheus, Loki, Alertmanager, tracing, capacity) and 8 write tools behind a flag. |
| **llm-worker** | `devops-ai-agent-worker` | SQS consumer in the private network. It bridges the agent to a private LLM and to GitHub Enterprise for GitOps PRs. Outbound-only. |
| **Helm chart** | `devops-ai-helm-charts` | Umbrella chart that deploys the three services as one release and refuses to render if their shared settings disagree. |
| **GitOps manifests** | `gitops-devops-ai-manifest` | Flux + Kustomize. The source of truth for what runs in each cluster. |
| **Sample apps** | `devops-sample-apps` | A four-service checkout stack with fault knobs, used to produce realistic incidents for testing and benchmarking. |

## Roadmap

| Item | Status |
|---|---|
| Multi-cluster, part 1: an SQS bridge so the MCP server can serve a cluster with no inbound network access | Done |
| Multi-cluster, part 2: route each investigation to the right cluster, and add a `cluster` label to incidents and memory | In design. A second (staging) cluster is being set up to validate it. |
| Multi-cluster, part 3: chart-managed bridges per cluster | Planned |

## Child pages

1. **Architecture** — components, network boundaries, data stores, cross-service contracts.
2. **How an Investigation Works** — from alert to RCA, step by step.
3. **Remediation and Safety Guardrails** — what the agent may change, and every check in front of it.
4. **Deployment and Configuration** — GitOps, the Helm chart, key settings, Slack and Alertmanager setup.
5. **Operations Guide** — dashboard, health, logs, tracing, troubleshooting.
6. **Slack User Guide** — how to work with the agent day to day.

## Glossary

| Term | Meaning |
|---|---|
| **RCA** | Root Cause Analysis — the structured answer the agent posts for an incident. |
| **MCP** | Model Context Protocol — the standard protocol the agent uses to discover and call tools. |
| **Tool** | One read-only (or, behind approval, write) operation such as `k8s_list_pods` or `prometheus_query`. |
| **Gate** | A deterministic check in code that can hold, rewrite or refuse what the model produced. |
| **Skill / playbook** | A Markdown guide for one kind of failure (e.g. OOMKilled), loaded into the model's context when the symptom matches. |
| **Light / heavy route** | The LLM router's two tiers: a cheap model for casual questions, a strong model for alerts and remediation. |
| **Approval card** | The Slack message with *Approve / Reject* buttons that every change must pass through. |
