---
title: "Agentic Ops"
order: 12
description: "The ops system is an autonomous agent with tiered guardrails and institutional memory."
---

## Agentic Ops

### 38. The Ops System Is an Autonomous Agent

The ops system is an autonomous agent, not a script runner. It follows the same architectural patterns as the main application — observe, reason, act, verify — but applied to infrastructure. The ops package is an app-within-the-app: its own server, database, routes, agent loop, and dashboard.

**The loop:** Observe cluster state → detect issues (LLM classification) → propose actions (LLM reasoning) → gate through guardrails → execute or queue for approval → verify outcomes → record everything.

**Tiered guardrails:**

- **Observe** (always allowed): read pods, logs, metrics, snapshots
- **Safe** (auto-approved): scale up, pull models, trigger jobs, take backups
- **Disruptive** (requires approval): restart pods, rollback deployments, scale down
- **Never** (hard-coded deny): delete PVCs, modify RBAC, alter ingress

**Every decision is recorded** with observations, reasoning, proposed actions, approval outcome, execution result, and rollback plan. The agent builds institutional memory — it learns what actions resolved what issues and proposes proven fixes first.

**The ops package mirrors the main app's patterns:**

```
Main App:  User query → intent detection → RAG retrieval → inference → response
Ops Agent: Cluster snapshot → issue detection → context retrieval → reasoning → action
```
