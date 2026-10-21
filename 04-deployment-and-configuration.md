# Deployment and Configuration

> **Audience:** engineers who deploy or change the stack. Parent: *DevOps AI Agent Stack — Overview*.

## How it is deployed

1. **Code repos** (agent, mcp-server, worker, sample-apps): a push to `main` runs GitHub Actions *Build & Push*.
   - It publishes `:latest` images.
   - The agent, mcp-server and worker images are amd64+arm64; the sample apps are arm64.
2. **`devops-ai-helm-charts`**: a `v*` tag publishes the umbrella chart to GHCR (OCI).
3. **`gitops-devops-ai-manifest`** (Flux + Kustomize):
   - `clusters/<cluster>/` — the Flux bootstrap for each cluster;
   - `apps/base/` — applications (devops-ai-stack, the sample apps, loadgen) and systems (postgres, prometheus, alertmanager, loki, fluent-bit, grafana, jaeger, cert-manager, longhorn, …);
   - `apps/dev/` — the dev overlay, the only overlay written so far.
4. **Flux** reconciles HelmRelease `devops-ai-stack` (chart 0.7.0) into namespace `devops-tools`.

- **GitOps is the source of truth.** A change to what runs in a cluster is a commit to `gitops-devops-ai-manifest`.
- **Rolling out new code.** Images are published as `:latest`. After a push to `main` and a finished image build, roll the Deployment so it pulls the new image:
  - `kubectl -n devops-tools rollout restart deploy/devops-ai-agent`;
  - the same command for `deploy/devops-mcp-server` and `deploy/devops-llm-worker`.
- **The agent reports its build.** It exposes the commit it was built from as `GIT_SHA`, which is also stamped on every recorded investigation.

## The umbrella chart

`charts/devops-ai-stack` packages the agent, the MCP server and the llm-worker as **one release**, with Postgres and Redis available as optional subcharts.

Its main job is `templates/_validate.tpl`, which **fails the render** when the services' shared values disagree:
- the MCP bearer token on the two sides;
- the SQS queue names on the two sides;
- the GitOps bridge settings.

A typo therefore fails in CI or in Flux's dry-run, instead of failing at 3 a.m. inside an incident. `MCP_HTTP_URL` is derived from the MCP server's Service rather than written twice.
- Shared settings live under `global:` (for example `global.sqs`, `global.aws`, and the switches for the bundled `postgresql` and `redis` subcharts).
- Each service has its own key. Its `env` and `secretEnv` always win over derived values.
- The chart **references Secrets but never creates them**.

## Key settings — devops-ai-agent

| Area | Setting | Notes |
|---|---|---|
| Slack | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` (Socket Mode), `SLACK_SIGNING_SECRET` | Secrets |
| Slack | `SLACK_ALLOWED_USERS`, `SLACK_APPROVER_USERS`, `SLACK_ONCALL_USERS` | Comma-separated Slack user IDs |
| Webhook | `ALERT_WEBHOOK_TOKEN` | Bearer token Alertmanager sends on `/alert`. If unset, the webhook is open (a warning is logged). |
| LLM | `LLM_PROVIDER` = `claude` / `openai-compatible` / `private-llm` / `router` | `router` holds several backends and picks one per call |
| LLM router | `LLM_BACKEND_<N>_NAME/_KIND/_MODEL/_KEY/_CONTEXT_TOKENS/_MAX_TOKENS/_PRICE_*`, `LLM_ROUTE_LIGHT`, `LLM_ROUTE_HEAVY` | Validated at boot: a typo is a pod that refuses to start |
| LLM | `LLM_MASK_IDENTIFIERS` | Default on |
| SQS | `SQS_REQUEST_QUEUE_NAME`, `SQS_RESPONSE_QUEUE_NAME`, `SQS_GITOPS_REQUEST_QUEUE_NAME`, `AWS_REGION` | FIFO queues; the response queue is shared and routed by `requestId` |
| MCP | `MCP_TRANSPORT=http`, `MCP_HTTP_URL`, `MCP_AUTH_TOKEN`, `MCP_TOOL_TIMEOUT_SECONDS` (45) | The token must match the MCP server's |
| Storage | `MEMORY_BACKEND=redis`, `REDIS_*`; `DB_HOST`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `DB_SSL_MODE` | Incident memory is disabled if `DB_HOST` is unset |
| Budgets | `MAX_CONCURRENT_INVESTIGATIONS` (5), `INVESTIGATION_TIMEOUT_SECONDS` (300), `MENTION_TOOL_ROUNDS` (2), `TOUR_TOOL_ROUNDS` (4) | |
| Delegation | `SUBAGENT_ENABLED`, `SUBAGENT_MAX_FANOUT` (3), `SUBAGENT_TOOL_ROUNDS` (3), `SUBAGENT_MAX_ITERATIONS` (5) | |
| Remediation | `GITOPS_REMEDIATION_ENABLED`, `REMEDIATION_VERIFY_DELAY_SECONDS` (300) | |
| Incidents | `INCIDENT_RECONCILE_ENABLED` and its `_MIN_AGE`, `_CONFIRM` and `_BATCH` settings | Closes incidents whose resolved webhook never arrived |
| Dashboard | `DASHBOARD_ENABLED`, `DASHBOARD_PORT` (3001), `DASHBOARD_PASSWORD`, `DASHBOARD_PUBLIC_URL` | Unset password ⇒ every page answers `503` |
| Tracing | `TRACE_ENABLED` (true), `LOG_LEVEL` | |

## Key settings — devops-mcp-server

| Setting | Notes |
|---|---|
| `TRANSPORT` = `http` / `stdio` | The agent connects over `http` in a cluster |
| `MCP_AUTH_TOKEN` | Required on `/mcp` |
| `K8S_AUTH_MODE=incluster` | Uses its ServiceAccount; the chart renders the ClusterRole (get/list, plus the write verbs when write tools are enabled) |
| `PROMETHEUS_URL`, `ALERTMANAGER_URL`, `LOKI_URL`, `TRACING_BACKEND` + `TRACING_URL` | The observability backends, each with optional basic auth |
| `MCP_ENABLE_WRITE_TOOLS` | Registers the 8 write tools |
| `ALLOWED_REMEDIATION_NAMESPACES` | Namespaces the write tools may touch. **Empty = all blocked.** |
| `MAX_SCALE_DELTA`, `MIN_ORPHAN_AGE_DAYS` | Bounds on scaling and orphan deletion |
| `UPSTREAM_TIMEOUT_SECONDS` | Upper limit on any upstream call |

## Key settings — llm-worker

| Setting | Notes |
|---|---|
| `SQS_REQUEST_QUEUE_NAME`, `SQS_RESPONSE_QUEUE_NAME`, `SQS_REQUEST_DLQ_NAME`, `SQS_MAX_CONCURRENCY`, `SQS_VISIBILITY_TIMEOUT_SECONDS` | Visibility timeout must exceed the slowest LLM call |
| `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`, `LLM_API_FORMAT` (`openai` / `anthropic` / `agent-builder`) | One worker deployment = one model; replicas of a model share its queue |
| `LLM_MAX_TOKENS`, `LLM_REASONING_EFFORT` | Reasoning models count hidden thinking toward the limit |
| `LLM_SOCKS_PROXY` | Optional SOCKS proxy for the LLM call only |
| `GITOPS_REPO`, `GITOPS_BRANCH`, `GITOPS_PATH_PREFIX`, `SQS_GITOPS_REQUEST_QUEUE_NAME` | The GitOps PR flow |
| `GITHUB_API_URL` + `GITHUB_TOKEN` (PAT) **or** `GITHUB_APP_ID` / `_INSTALLATION_ID` / `_PRIVATE_KEY(_FILE)` | GitHub Enterprise credentials, held only here |

## AWS authentication (SQS)

| `AWS_AUTH_MODE` | Use |
|---|---|
| `iam-anywhere` | On-premise or private network with an X.509 certificate |
| `irsa` | EKS, IAM Roles for Service Accounts |
| `instance-profile` | EC2 / ECS |
| `env` | Local development and CI only |

## Slack app setup

1. Create the app at api.slack.com/apps.
2. Bot scopes: `app_mentions:read`, `chat:write`, `channels:history`, `groups:history`, `reactions:read`.
3. Event subscriptions: `app_mention`, `reaction_added`.
4. Enable **Socket Mode** and create an app-level token with `connections:write`.
5. Enable **Interactivity** for the Approve/Reject buttons. Over Socket Mode no request URL is needed.
6. Invite the bot to the alert channel.

## Alertmanager receiver

```yaml
receivers:
  - name: devops-ai-agent
    webhook_configs:
      - url: http://devops-ai-agent.devops-tools:3000/alert
        send_resolved: true            # enables the resolved-alert loop
        http_config:
          authorization:
            credentials: <ALERT_WEBHOOK_TOKEN>
route:
  group_by: ["alertname", "namespace"]   # one webhook per group → one investigation
  repeat_interval: 12h
  receiver: devops-ai-agent
```

Adjust the URL to the agent Service name in your cluster.

## Database

- Postgres needs a database named in the release values (`auth.database`, e.g. `devops_agent`). The Bitnami chart creates it only on the **first** initialisation of an empty volume. An already-initialised volume needs a one-time manual `CREATE DATABASE`.
- Tables are created by the agent's own migrations at startup (`001`–`015` today). A failing migration is a pod that does not start, so migrations avoid statements that need extra privileges (no `CREATE EXTENSION`).

## Changing behaviour without a rebuild

- **System prompt**: `prompts/system.md`.
- **Playbooks**: `prompts/skills/*.md`. Each file has flat `name`, `description` and `when` frontmatter, where `when` is `always` or a regex over the alert text. A malformed skill makes the pod refuse to start, and the test suite loads the same directory, so a bad file fails CI first.
