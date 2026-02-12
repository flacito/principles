---
title: "Capabilities & Composition"
order: 1
description: "The foundational unit of work and how capabilities compose into systems."
---

## Capabilities & Composition

### 1. Capability as the Unit of Work

A capability is a single, discoverable operation with a clear input, a clear output, and a bounded purpose. It is the atomic unit of the system — not a file, not a class, not an endpoint. Files implement capabilities. CLIs expose them. MCP servers advertise them. Agents invoke them. If you can't describe what a capability does in one sentence without the word "and," it's two capabilities.

Capabilities are stateless by default. Dependencies like database connections or service clients are created on demand, used, and released. When a capability _must_ hold a persistent connection, it is an exception that carries an obligation: the capability must expose a health check in its contract so that a watcher capability can verify the connection is alive. An unwatched stateful capability is not permitted.

### 2. Composition Over Orchestration

Capabilities compose into larger capabilities. A CLI composes capabilities into a user-facing tool. An agent composes capabilities by discovering and invoking them through protocol. A pipeline is itself a capability — one whose purpose is to invoke and watch other capabilities in a deterministic order. There is no special class of "orchestrator." Every layer follows the same rules: bounded purpose, clear contract, one sentence without "and."

No capability should assume it will only ever be called as part of a sequence. It must be invocable in isolation with a meaningful result, even if its most common use is inside a pipeline.

### 3. Protocol Is the Interface

Capabilities communicate over standard protocols, not custom APIs. MCP over stdio is the default: the capability is a process, the contract is JSON-RPC, and the transport is stdin/stdout. This is local, air-gap compatible, and composable by any caller — human CLI, agent, or another capability.

HTTP/SSE is the exception, used when a capability must be shared across a network boundary. The capability's logic does not change — only the transport. A capability that only works over HTTP or only works over stdio has a transport dependency baked into its logic and needs to be refactored.

Custom REST APIs are a last resort for browser-facing boundaries where protocol translation is unavoidable. Even then, the REST layer is a thin adapter over an MCP capability — not a place where logic lives.

### 4. Every Capability Declares Its Contract

A capability must be self-describing. Its name, purpose, inputs, outputs, and observability are part of its definition — not documented elsewhere, not implied by reading the source. An agent encountering a capability for the first time has everything it needs to decide whether to invoke it, how to invoke it, and how to watch it.

The contract includes how the capability should be observed. A short-lived capability declares its exit codes and log format. A long-running capability exposes health and readiness. A pipeline capability reports status of its children. Observability is not an ops concern added later — it is part of the capability's identity from inception. A capability that cannot be watched cannot be trusted in composition.

In MCP, this means every tool has a schema: a name that reads as a verb-noun action, a description that passes the one-sentence test from Principle 1, and Zod-validated input/output shapes. In CLI form, this means `--help` is comprehensive and the command fails loudly with actionable guidance when inputs are wrong. In both cases, the contract is the capability — if the schema is unclear, the capability is unclear.

### 5. Converge, Don't Assume

A capability never assumes its environment is ready. It checks, reports, and guides — whether the caller is a human or another agent. This is the "single command to satisfaction" principle applied universally: run, discover what's missing, report what's needed, wait, repeat until converged.

When an agent invokes a capability and it cannot proceed, the response is not an opaque failure. It is a structured description of what is missing and how to resolve it. A calling agent can act on that description — install a dependency, start a service, provision a resource — and retry. This creates a convergence loop: invoke → diagnose → remediate → invoke → succeed. Two agents communicating over stdio can drive a system from broken to running without human intervention, so long as each capability honestly declares what it needs and what it found.

The `[OK]` / `[SKIP]` / `[TODO]` / `[!!]` pattern from idempotent scripts is the human-readable form of this contract. The machine-readable form is structured JSON over MCP with the same semantics: done, already done, needs this, or broken in this way.

A capability that fails with an opaque error message has violated its contract. The failure must be actionable. "Connection refused" is not actionable. "ChromaDB is not running on localhost:12100. Start it with `docker compose up chromadb` or set CHROMA_URL to an alternative host" is actionable. The capability does not fix the problem — it diagnoses with enough precision that the caller, human or agent, can.

### 6. Every Capability Declares Its Blast Radius

A capability's contract must include its risk tier. What can this capability affect, and is the effect reversible? The tiers are:

- **Observe** — reads state, changes nothing. Always safe to invoke.
- **Safe** — creates or adds. Reversible or idempotent. Low risk to auto-approve.
- **Disruptive** — modifies or restarts existing state. Requires explicit approval or a caller with authority.
- **Destructive** — deletes, revokes, or otherwise performs an irreversible action. Must never auto-execute. Must require confirmation even from an authorized agent.

The risk tier is not documentation. It is part of the machine-readable contract — an agent can filter, gate, and audit based on it. A capability that does not declare its blast radius is treated as destructive by default. Silence is not safety.

### 7. The System Knows Its Own Risk Posture

Every MCP composition must include a meta-capability that can enumerate all registered capabilities, their contracts, and their risk tiers. This produces a system risk profile — a complete inventory of what the system can do, what is dangerous, and what is exposed.

Before an agent executes a plan involving multiple capabilities, it can query this inventory, compute a composite risk score, and present it for acceptance. A run that composes three observe capabilities and one safe capability is routine. A run that includes a disruptive capability requires acknowledgment. A run that includes a destructive capability requires explicit risk acceptance with an audit trail.

This is not optional infrastructure — it is a first-class capability in every MCP server, present from day one.

### 8. Human in the Loop Is the Default

No capability above the observe tier executes without human awareness. This is the default posture — it can be relaxed, but only deliberately, informed by risk, and with audit.

Every MCP server exposes a **human interaction context** — a contract that defines the current autonomy boundary. This context is derived from the system risk profile (Principle 7) and declares which tiers the agent may auto-execute and which require human approval. The default context is conservative: observe is autonomous, everything else asks.

A human can widen the autonomy boundary at any time. An agent may _request_ wider autonomy by presenting the specific capabilities it wants to run freely, their risk tiers, and the composite risk score. The human grants, narrows, or denies. When a human says "GO GO GO," the system captures exactly what was granted, at what risk level, by whom, and when. The decision is documented, the accountability is clear, and the grant is scoped — not a blank check.

Autonomy grants expire. A session-level grant ends with the session. A persistent grant has a TTL and must be renewed. An agent that was trusted yesterday is not trusted today without a living grant.

When a human is asked to approve or widen autonomy, the system presents risk in plain language — not a legal disclaimer, but a genuine, concise statement of what could go wrong and whether it is reversible. The human's job is to judge risk. The system's job is to make sure they have what they need to judge well.

### 9. The Binary Is a Composition of Capabilities

A CLI binary is not a special construct. It is a capability that composes other capabilities behind a command interface. `skjv --serve` invokes the serve capability. `skjv --chat` invokes the chat capability. The binary is a surface, not a brain — it parses arguments, resolves the requested capabilities, and delegates. Logic does not live in the CLI layer.

Binaries follow the same single-purpose test as any capability. A binary that tries to serve every use case is a monolith hiding behind a flag parser. The art is in deciding what coheres into one binary and what deserves its own. The test: do these capabilities share a _user identity_? `skjv` is the Bible scholar's tool. `skjv-ops` is the operator's tool. `skjv-deploy` is the release engineer's tool. Different humans, different binaries — even if the underlying capabilities overlap.

Every binary is a Node.js Single Executable Application (SEA). It runs without requiring the user to have Node installed, a repo cloned, or dependencies resolved. One file, one purpose, runs anywhere.

### 10. Single Command to Convergence

Every entry point — binary, capability, or agent — either succeeds or produces a structured path to success. This is not error handling. This is the operational contract: run, assess the environment, report what's present, report what's missing, guide toward resolution, and be safe to run again.

The convergence vocabulary is universal across human and machine callers:

- `[OK]` — action completed successfully
- `[SKIP]` — already done, nothing to do
- `[TODO]` — needs intervention, here's specifically what and how
- `[!!]` — something is broken, here's what to check

For human callers, this is terminal output. For agent callers, this is structured JSON over MCP with the same four semantics. An agent receiving a `[TODO]` can act on it — install a dependency, start a service, adjust a configuration — and retry. Two agents communicating over stdio can converge a system from broken to running without human intervention, each cycle resolving one `[TODO]` until only `[OK]` and `[SKIP]` remain.

A capability that fails with an opaque error message has violated its contract. The failure must be actionable. "Connection refused" is not actionable. "ChromaDB is not running on localhost:12100. Start it with `docker compose up chromadb` or set CHROMA_URL to an alternative host" is actionable. The capability does not fix the problem — it diagnoses with enough precision that the caller, human or agent, can.

### 11. Air-Gap First

Every capability is designed to function without network access. This is not a deployment afterthought — it is a design constraint that shapes every decision. Dependencies are local. Data is local. Models are local. Protocol defaults to stdio, not HTTP. External services are optional enhancements, never requirements.

Air-gap first forces honesty in architecture. A capability that silently depends on a remote service will fail in the field with no explanation. A capability designed for air-gap declares all its dependencies in its contract (Principle 4), checks for them at invocation (Principle 10), and degrades gracefully when optional services are absent (Principle 36: Progressive Degradation).

When network access is available, capabilities may use it to enhance results — pulling updated models, syncing data, reaching cloud APIs. But the core function works without it. The system is useful in a disconnected submarine, a classified SCIF, or a developer's laptop on an airplane. Network is a boost, not a crutch.

### 12. Environment Is Not Special

A capability runs the same way everywhere. The laptop, the staging cluster, the air-gapped SCIF, and production are not different systems — they are the same topology at different scales. Same containers, same services, same ports, same data flow, same convergence contract.

This means no "dev mode" shortcuts that diverge from production behavior. A capability's contract declares its dependencies. The convergence loop (Principle 10) provisions them locally or in a cluster using the same logic. If a capability requires an environment-specific code path to function, it has a design problem — its dependencies are not properly declared or its contract is lying.

A developer's laptop is a datacenter with a single node. A production cluster is the same datacenter with many. The single command that starts the full runtime locally is the same command that deploys to production with a different target. One topology, one contract, scaled up or down.
