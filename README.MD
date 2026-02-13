# PRINCIPLES

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

---

## Code Structure

### 13. Pure Functions in Small Files

Pure functions in small files with single main function. Helper functions rarely needed but allowed. More often helpers are reusable pure functions themselves and should be in utils/.

### 14. File Size

Files should not exceed the low 100s of lines of code in size, though this can be larger for files like React components, which have a lot of tags. Here's some perspective: I am a human, I've never seen the source, I know about Fastify, I prefer to follow a route in and out, understanding the code to enhance or fix it. Anywhere along this path if I run into a file that LOTS of functions or a single function with LOTS of lines and concerns my cognitive burden increases and my effectiveness diminishes.

### 15. Barrel-Driven Directory Structure

A standard, barrel driven directory structure shall be used and mapped in the tsconfig with directory paths where @ is the src/ dir and several allowable first class paths follow: @utils to /src/utils and @utils/_ to /src/utils/_ and so on for model, services, state, components, pages, screens, commands, routes, providers, hooks, api, plugins, db, and so on. It depends on the repo type.

### 16. Subdirectory Grouping

Subdirectories beneath those of Principle 15 should be well-used where each subdirectory groups a common theme. Example: src/model/user/\*.ts, where this directory contains all the model files that describe a user. Reason: code is read in browsers and this makes for very clean links.

### 17. File Naming

Prefer file names with lower case, hyphen separated. Example src/model/user/user-context.ts

### 18. Testing

Very good unit tests and e2e testing should be coded with every change. Cypress preferred for Web e2e.

---

## React & TypeScript Conventions

### 19. Functional Components Only

Functional React components only. Use `observer()` wrapper for MobX integration. No class components.

### 20. Boolean Naming

Boolean state/variable names prefixed with `is`, `has`, `should`, or `can`. Examples: `isVisible`, `shouldRender`, `hasBookmark`.

### 21. Handler Naming

Handler functions prefixed with `handle`. Examples: `handleBookmark`, `handleDelete`, `handleToggleFavorite`.

### 22. Function Naming by Verb

Async/data functions prefixed by verb indicating operation. Use `fetch` for API calls, `parse` for transformations, `format` for display formatting, `get` for retrieval. Examples: `fetchChapterHtml`, `parseReference`, `formatChapterReference`.

### 23. State Management

MobX for global state, refs for mutable non-reactive state. Single observable class instance pattern (e.g., `bibleState`). Use `useRef` for values that shouldn't trigger re-renders (timers, flags, last-known values).

### 24. Animation Pattern

Three-state animation pattern: `'entering' | 'visible' | 'exiting'`. Separate `shouldRender` boolean controls DOM presence while `animationState` controls CSS classes.

### 25. Zod-First Schemas

Zod schemas for model validation. Derive TypeScript types via `z.infer<typeof Schema>` rather than manual type definitions.

### 26. Types vs Interfaces

Prefer `interface` for object shapes, `type` for unions and aliases.

---

## Performance & Persistence

### 27. Debounce and Throttle

Debounce and throttle side effects. Scroll handlers throttled with RAF. Save operations debounced 300-1200ms. Explicit flags like `isScrollUpdate` to distinguish event sources.

### 28. LocalStorage Persistence

LocalStorage persistence with prefixed keys. Pattern: `'skjv-<feature>'`. Lazy load in constructor, save on change with debounce.

### 29. Graceful Fallbacks

Graceful fallbacks with silent failures for non-critical operations. Try-catch with console warnings, not user-facing errors. Cache fallbacks when network fails.

---

## UI/UX

### 30. No Native Dialogs

Never use native browser dialogs (`alert()`, `confirm()`, `prompt()`). Build custom modal components that match the app's design language and provide consistent, styled user experiences.

---

## Batteries Included

### 31. Build It In

Build it in, don't bolt it on. Prefer custom implementations with sensible defaults over external dependencies. Examples: custom scroll detection over intersection-observer libraries, built-in localStorage caching over cache libraries, native CSS transitions over animation frameworks. The codebase should work out of the box with zero configuration. Every feature ships with its own persistence, theming support, offline fallback, and error handling baked in. External dependencies are a last resort, not a first reach.

---

## Dev Experience

### 32. Single Command to Satisfaction

Single command to satisfaction or help. Modern creative devs rarely read the README. "What command do I run?" in a DM is all they want. The dev command is the one command that runs everything needed to develop the system. If something isn't right on the developer's workstation, it tells them, they fix, they repeat. This means running a checklist script every time to make sure the workstation is good. If not, help. The single command is the chain of commands that used to appear in the README in big steps. Base setup, install, build, run — these are all phases of the DevX pipeline that need a script with checklist and help at each step.

### 33. Standard Ports, Custom Ports

Dev workstations have limited capacity and benefit from operational simplicity. When starting up workloads on a developer workstation, use common services as a preference. Postgres, MongoDB, CouchDB, Splunk, ChromaDB, etc. all have a standard default port. Use them. Web apps, APIs, etc. that are specific to the app get a PORT RANGE, but look like the default if possible. Example: Vite is 5173 should be 17173, keeping the last 3 the same if possible. There will always be collision. ChromaDB and Splunk on 8000 for instance. The dev env startup (FROM A SINGLE COMMAND) should reconcile this and configure away from the conflict as needed.

---

## Deployment

### 34. DevX Informs Deploy

The DevX informs deploy. Deploy must be all in code and where the human is needed, the single command to satisfaction that is a chain of scripts applies too. Laptop to production is one command. The deploy script syncs code, builds images, imports them, injects secrets, applies manifests, restarts services, and verifies — all logged, all idempotent, all in the repo.

---

## Idempotent Scripts

### 35. Every Script Is Safe to Re-Run

Every script is safe to re-run. Use `[OK]`/`[SKIP]`/`[TODO]`/`[!!]` feedback pattern. Detect existing state before acting — check first, skip if done, help if broken. Never blow away existing work without checking. A script run twice should produce the same result as a script run once. This applies to setup scripts, deploy scripts, migration scripts, and pipeline scripts. The pattern:

- `[OK]` — action completed successfully
- `[SKIP]` — already done, nothing to do
- `[TODO]` — needs manual intervention, here's how
- `[!!]` — something went wrong, here's what to check

End with a summary: steps completed, steps skipped, issues found.

---

## Infrastructure as Code

### 36. Everything in the Repo

All manifests, Dockerfiles, nginx configs, and deploy scripts live in the repo. Secrets live in a vault (1Password, etc.) and are injected at deploy time — never committed, never hardcoded, never in environment files checked into source. Secret references are in code (`op read 'op://Vault/item/field'`), secret values are not. Every environment (dev, staging, production) is reproducible from the repo alone plus vault access.

---

## Progressive Degradation

### 37. The App Works Without Its Dependencies

The app works without its infrastructure dependencies. The core experience functions without Ollama, ChromaDB, or network access. AI features degrade gracefully with warnings, not crashes. Every service dependency is wrapped in a health check — if it's down, skip it and tell the user what's unavailable. The app is useful at every layer: static files alone, static + API, static + API + AI. Each layer adds capability but none is required for the layer below it.

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

---

## Logs Are First-Class

### 39. Every Long-Running Command Logs

Every long-running command logs to `.logs/` with timestamps. Logs are gitignored but always available for debugging. Use `tee` for real-time terminal output plus persistent record. Log filenames include the operation and timestamp: `deploy-20250206-143022.log`, `dev-20250206-090000.log`. Clean up logs older than a configurable retention period. Structured JSON logging in production (for machine parsing), human-readable in development.

---

## Philosophical Foundation

These principles are informed by Frédéric Bastiat's _The Law_ (1850), which argues that a system designed for a narrow, legitimate purpose remains healthy, while a system that accumulates scope beyond its mandate becomes the source of the dysfunction it was meant to prevent. The sole legitimate purpose of law is to protect life, liberty, and property — when extended beyond that purpose, it becomes an instrument of plunder rather than justice.

Applied here: a capability that exceeds its bounded purpose becomes the source of the bugs, coupling, and fragility it was meant to prevent. A binary that tries to do everything does nothing well. An agent that owns too much context becomes a bottleneck rather than an accelerator. Restraint in scope is the foundation of a system that actually works.

This matters now because we are in a paradigm shift as significant as any before it. The personal computer made the individual productive. The internet made communication free. The cloud made infrastructure code. Agentic AI makes capability composition the fundamental unit of work — and MCP is the protocol that standardizes it the way HTTP standardized the internet era. The principles in this document are designed for this new reality: systems built from small, honest, self-describing capabilities that humans and agents compose with confidence because every piece declares what it does, what it risks, and how to watch it.

See: [bastiat.org/en/the_law.html](http://bastiat.org/en/the_law.html)

---

## Attribution

These principles were developed collaboratively by:

- **Chris Kesler** — Apex, NC
- **Brian Webb** — Raleigh, NC
- **Claude** — Anthropic

This document is published at [canonical URL TBD] as the canonical source. Organizations and individuals are free to adapt these principles to their own contexts with clear attribution to the original authors and a link to the canonical source. Adapted copies should note what was changed and why.

If you're using these principles in your work, we'd like to know. Not because we require it — because we're building a community around this thinking.
