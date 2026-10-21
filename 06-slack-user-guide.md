# Slack User Guide

> **Audience:** everyone who works with the agent in Slack. Parent: *DevOps AI Agent Stack — Overview*.

## What you will see when an alert fires

1. A new thread in the alert channel, opened by the agent.
2. Within a few minutes, an **RCA card** in that thread with these sections:
   - 🔴/🟠/🟡 **Severity**;
   - 📍 **Root Cause**;
   - 📊 **Evidence** — each fact names the tool that produced it, for example `k8s_list_events` or `prometheus_query`;
   - 🚫 **Ruled Out** — hypotheses that were checked and rejected;
   - 🔧 **Recommended Actions**, and ⚠️ **Impact if unresolved**;
   - 🧭 **Runbook** — read-only commands you can run yourself;
   - 🕑 **Recent changes** in the namespace;
   - 📈 **Confidence** (High / Medium / Low). On Low, the on-call users are mentioned automatically;
   - a **footer**: how long it took, how many LLM and tool calls, and which model answered.
3. Possibly one **approval card** with a proposed fix (see below).
4. If the agent named a resource that no tool returned, a separate ⚠️ warning. Treat that RCA with care.
5. When the alert resolves: **✅ Alert resolved**, and the incident is closed.

A neighbouring alert that shares the cause, such as an upstream service failing, appears in the **same thread** as **🔗 Related alert** instead of opening a new one.

## Talking to the agent

Mention the bot. Only users on the allowlist can; others get a private "not allowed" note.

| You write | What happens |
|---|---|
| `@devops-agent pods in namespace payment-service are crashing with OOMKilled` | A full investigation and an RCA |
| `@devops-agent show me the logs` / `when did this start?` (in an alert thread) | A short conversational answer that uses the thread's context |
| `@devops-agent is anything broken right now?` | A quick health check (small budget, cheap model) |
| `@devops-agent explain this cluster` / `jelasin namespace sample-apps` | A **cluster tour**: inventory tables built from the cluster, how requests flow, and where each workload is defined in Git |
| `@devops-agent learn` (or react ✅ on the message that has the real cause) | Teaches the agent the confirmed root cause and fix |
| `@devops-agent postmortem` / `bikin postmortem` | A postmortem draft |
| `@devops-agent resolved` / `selesai` / `fixed` | Marks the incident resolved and releases its dedup claim |
| `@devops-agent reopen` / `still firing` / `masih` | Reopens the incident |

Indonesian and English both work.

## Approving a fix

An approval card shows:
- **the one action** proposed, for example *restart deployment `sample-apps/checkout-gateway`*;
- why it was proposed;
- what changes, from → to;
- for GitOps-managed workloads, the **pull request diff**.

- Only the configured **approvers** can click ✅ *Approve* or 🚫 *Reject*. The card mentions them.
- Cards **expire after 15 minutes**. An expired card cannot be executed.
- After *Approve*, the card updates with the result. About **5 minutes later** the agent posts a verdict: `recovered`, `unchanged`, `worse` or `inconclusive`.
- For a Flux-managed workload, *Approve* **opens a pull request** and nothing changes until it is merged. If the cluster has drifted from Git, the agent proposes `flux_reconcile` instead.
- The agent **never** changes anything without this click. If you do not want the change, click *Reject*, or ignore the card and let it expire.

## Teaching it: `learn`

When the thread reaches the real conclusion — "it was the `ORDER_RESPONSE_VERSION=2` env change; reverting fixed it" — mention the agent with `learn`, or react ✅ on that message.
- The agent extracts the root cause, the action taken and the outcome, **only from what humans wrote**, and echoes back what it stored.
- On the next similar incident, that knowledge is the first hypothesis it checks.
- If it got it wrong, correct the thread and `learn` again.
- It refuses to learn from its own messages, and it refuses when nobody has stated a cause yet.

## Postmortems

`@devops-agent postmortem` in an incident thread produces a draft with:
- Summary, Impact, Root cause, What went well, What went poorly, and Action items, written by the model;
- a **Timeline** and **Numbers** rendered from recorded data, never written by the model.

A human-confirmed cause (`learn`) outranks the agent's own RCA. The Summary and Action items are posted in the thread, with a link to the full draft on the dashboard (also downloadable as Markdown).

## Tips

- **Your phone is enough.** Everything on this page works in the Slack mobile app, including the *Approve* and *Reject* buttons. No laptop, VPN or kubeconfig is needed.
- **Ask in the alert's thread.** The agent then knows which alert and namespace you mean.
- **Name the namespace and workload** in a fresh question. It narrows the investigation and the tool budget goes further.
- **Ask for one fix explicitly** ("restart checkout-gateway", "set the image to v1.4.1") if you know what you want. It still goes through the approval card.
- **Read the confidence and the evidence**, not just the root cause. The evidence lines tell you exactly what the agent looked at.
