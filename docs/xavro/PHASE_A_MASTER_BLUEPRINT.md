# PHASE A — XAVRO AI: COMPLETE MASTER BLUEPRINT

> Status: planning document, version 1.0 (2026-10-08). No application code exists yet.
> Phase B (`PHASE_B_MASTER_BUILD_PROMPT.md`) is derived from this document and is the implementation directive.

## How to read this blueprint

Every technology claim in this document carries a maturity label:

| Label | Meaning | Policy |
|---|---|---|
| **STABLE** | Generally available, documented, in broad production use, with a compatibility promise. | Safe for the critical path. |
| **PREVIEW** | Vendor-released but labelled alpha, beta, preview or release candidate; APIs may still change. | Allowed only behind an internal interface, with a fallback. |
| **EXPERIMENTAL** | Published and usable but young, fast-moving, or with a small production footprint. | Allowed only off the critical path, behind a feature flag. |
| **UNPROVEN** | Claimed by vendors or commentators but not independently verified, or no meaningful production evidence. | Not used. Tracked for re-evaluation. |

Where this document cites numbers from vendors (benchmarks, latency, cost), they are labelled as vendor claims. Fast-moving facts were checked against sources in October 2026 and must be re-verified at implementation time (see §35).

---

## 0. Research summary: the landscape in October 2026

This section records what was learned before any design decision was made. It is the evidence base for the rest of the blueprint.

### 0.1 Coding agents and AI workspaces

The market has split into three execution models:

1. **Local terminal agents** (Claude Code, OpenCode, Aider, Hermes): run on the developer's machine with full shell access. Strength: real environment, fast loop, rich tool use. Weakness: requires a computer; safety depends on the user's machine and their attention to permission prompts.
2. **IDE agents** (Cursor, Windsurf, Google Antigravity, GitHub Copilot, Cline): agent inside an editor. Strength: tight edit/review loop, excellent diff views. Weakness: desktop-only; the editor is the product, so the agent is a feature of it.
3. **Cloud / background agents** (OpenAI Codex cloud tasks, Devin, Google Jules, Copilot coding agent): the agent runs in a remote sandbox and returns a branch or pull request. Strength: asynchronous, runs while the user is away, works from any device. Weakness: weak mid-run visibility and steering, opaque environment, coarse permission models, and review experiences that are designed for desktop browsers.

Observed patterns worth learning from (not copying):

- Parallel sub-agents and long autonomous runs are now normal at the top end; benchmark leaders (vendor-reported SWE-bench Verified and Terminal-Bench figures in the 80% range) are differentiated more by **harness quality** (context gathering, verification, tool design) than by raw model choice.
- **Quota exhaustion and unpredictable cost** is the most common complaint across products. Users do not know what a task will cost before it starts.
- **Broad, ambiguous prompts produce poor autonomous results.** The products that do well force a clarification or planning step for underspecified tasks.
- **Approval fatigue** is the main failure mode of approval-gated agents (Cline-style). Users either rubber-stamp or abandon.
- **No product treats the phone as a first-class surface for agent work.** Mobile access, where it exists, is a responsive web page or a chat app with a link out. Reviewing a diff, steering a running agent, approving a risky step, or reading a terminal log on a phone is consistently poor.

### 0.2 Protocols

- **Model Context Protocol (MCP)** — STABLE for core tool/resource calls. The **2026-07-28 specification** makes the core protocol stateless (no `initialize` handshake, no session id, any instance can serve any request, routing via an `Mcp-Method` header), promotes **Tasks** to a first-class extension for long-running operations with pollable/resumable state, hardens OAuth/OIDC authorization, requires tool input schemas to be full **JSON Schema 2020-12**, introduces **MCP Apps** (server-rendered sandboxed UI), allows `tools/list` TTL caching, and deprecates Roots, Sampling and Logging (in favour of standardized tracing keys). Ecosystem adoption of the new revision is in progress, so the new features are **PREVIEW** in practice until major SDKs and servers ship them.
- **Agent2Agent (A2A)** — governed by the Linux Foundation, reports 150+ participating organizations and enterprise production use in its first year. Useful for agent-to-agent interoperability between organizations; **EXPERIMENTAL** for Xavro's purposes because Xavro's own agents are internal and do not need a cross-vendor wire protocol in early phases.

### 0.3 Agent security

The **OWASP Top 10 for Agentic Applications (2026)** is the reference threat taxonomy used in this blueprint:

| ID | Risk |
|---|---|
| ASI01 | Agent Goal Hijack (objective steered by content the agent reads) |
| ASI02 | Tool Misuse & Exploitation |
| ASI03 | Identity & Privilege Abuse (over-broad or borrowed credentials) |
| ASI04 | Agentic Supply Chain Vulnerabilities (frameworks, connectors, MCP servers loaded at runtime) |
| ASI05 | Unexpected Code Execution |
| ASI06 | Memory & Context Poisoning |
| ASI07 | Insecure Inter-Agent Communication |
| ASI08 | Cascading Failures |
| ASI09 | Human-Agent Trust Exploitation (manipulating approvals) |
| ASI10 | Rogue Agents |

The key lesson: **prompt injection cannot be reliably prevented by the model**. It must be contained by architecture: least-privilege credentials, isolation, egress control, and approvals that show ground truth rather than the agent's own description of what it is about to do.

### 0.4 Execution sandboxes

- **Firecracker microVMs** — STABLE. Own guest kernel under KVM; sub-125 ms boot and under 5 MiB overhead (project documentation); snapshot/restore enables warm pools; syscall-heavy agent workloads (git, package installs, test runs) run at native speed inside the guest. Requires `/dev/kvm` (bare metal or nested virtualization).
- **gVisor** — STABLE. User-space kernel intercepting syscalls; runs on any Linux host and plugs into Docker/Kubernetes; per-syscall overhead compounds on I/O-heavy agent loops.
- **Kata Containers** — STABLE. MicroVM boundary inside Kubernetes (can use Firecracker underneath).
- **Managed agent sandboxes** (E2B, Daytona, Modal, Beam and others) — STABLE to PREVIEW depending on vendor; they sell microVM- or gVisor-class isolation with an API. They remove months of infrastructure work at the cost of per-second pricing and vendor dependency.
- Neither Firecracker nor gVisor protects against hardware side channels; both depend on the surrounding design (credentials, reachable network, teardown).

### 0.5 Durable execution (long-running, resumable agent runs)

- **Temporal** — STABLE, the most proven. Four core services plus an external database (or Temporal Cloud, billed per action). Workflow/Activity split; long sessions need Continue-As-New.
- **Restate** — STABLE (younger). Single binary with an embedded replicated log, durable functions, workflows, durable RPC, **Virtual Objects** (per-key state with built-in single-writer concurrency), awakeables for external signals, Kotlin/Java/TypeScript/Go/Python/Rust SDKs, self-host or BYOC. Vendor-claimed sub-10 ms P99 per durable step.
- **DBOS** — STABLE (young). Durable workflows as a library on top of PostgreSQL; minimal infrastructure.
- Home-grown state machine on PostgreSQL — viable but reinvents retries, timers, idempotency and replay.

### 0.6 Android platform

- **Jetpack Compose** + **Material 3** — STABLE.
- **Compose Material 3 Adaptive 1.3.0** — STABLE (2026-08-12): `ListDetailPaneScaffold`, `SupportingPaneScaffold` (with margins and edge-to-edge), `adaptive-navigation3` scene strategies. `NavigationSuiteScaffold` exists; component-override APIs are changing in 1.4.0-alpha (PREVIEW). Window-size helpers have been reshaped (`currentWindowAdaptiveInfo` V2), so code must isolate them.
- **Jetpack Navigation 3** — STABLE (since November 2025). Back stack is plain state owned by the app; integrates with adaptive scenes.
- **Android 16 large-screen rules** — apps targeting API 36 have orientation, resizability and aspect-ratio restrictions ignored on large screens (≥600 dp). Every screen must be genuinely adaptive; locking to portrait is no longer an option on tablets and foldables. Desktop windowing on tablets is a real usage mode.
- **On-device Gemini Nano via ML Kit GenAI APIs (including the Prompt API)** — PREVIEW (Prompt API released as alpha in October 2025), and limited to supported devices. Useful only for small private tasks (summarizing a notification, drafting a title), never for the agent.
- **Credential Manager** — STABLE. Unified passkeys, passwords and Sign in with Google on Android; passkeys are Google's recommended primary credential.

### 0.7 Data, retrieval and code intelligence

- **PostgreSQL** — STABLE. With **pgvector** (STABLE, HNSW indexes) it covers transactional data plus vector search to the low tens of millions of vectors with acceptable latency. Dedicated engines (**Qdrant** STABLE, **Turbopuffer** STABLE managed, object-storage-backed and cheap at very large scale, Pinecone STABLE managed) win at very large or multi-tenant-heavy scale.
- **tree-sitter** — STABLE. Incremental parsing for 100+ languages; the de facto basis for syntax-aware chunking, highlighting and symbol extraction.
- **SCIP** (Sourcegraph Code Intelligence Protocol) indexers — STABLE for TypeScript, Java/Kotlin, Python, Go and others; give precise definitions/references when the project builds.
- **ripgrep** — STABLE. Still the fastest way for an agent to search a working tree; in published agent harnesses lexical search plus reading files outperforms embedding-only retrieval for code.

### 0.8 Observability and evaluation

- **OpenTelemetry** — STABLE for traces, metrics, logs. The **GenAI semantic conventions** (model calls, agent and tool spans such as agent invocation and tool execution, MCP conventions) have moved to a dedicated repository (`open-telemetry/semantic-conventions-genai`) and remain in **development** status; attribute names may change. Use them, but pin a version and map through one internal module.
- LLM-native tracing/eval platforms (**Langfuse** open-source and OTel-native, others commercial) — STABLE.
- Agent evaluation harnesses (SWE-bench family, Terminal-Bench, UK AISI **Inspect**) — STABLE as tools; benchmark scores are vendor-reported and saturating, so product-specific evaluation suites are required.

### 0.9 GitHub

- **GitHub Apps** — STABLE and the recommended integration model: fine-grained per-permission scopes, per-repository installation, **installation access tokens that expire after one hour** and can be **down-scoped to specific repositories and permissions** at mint time, user-to-server tokens when actions must be attributed to the user, and webhooks. Commits created through the API by an app are signed and shown as verified. Personal access tokens and OAuth Apps are the wrong model for an agent product (broad, long-lived, user-managed).

### 0.10 What the research says Xavro must not assume

- That a model can be trusted to obey a permission described in its prompt. (It cannot; enforcement must be outside the model.)
- That one model provider will stay best. (Leadership changed several times in 2025-2026.)
- That embeddings solve code retrieval. (They help ranking; they do not replace search and reading.)
- That a phone can run the agent. (It cannot run builds, tests or large models; the phone is the **control surface**, the cloud is the **workbench**.)
- That users want to type code on a phone. (They want to **direct, review, approve and understand** on a phone; editing is occasional and short.)
- That autonomous runs are cheap. (Cost and quota transparency is a product feature, not billing plumbing.)

---

## 1. Product foundation

### 1.1 Product vision

**Xavro is the place where you think with AI and then hand it real work — and stay in control of that work from your pocket.**

It is a workspace, not a chatbot: conversations, projects, repositories, files, a cloud workbench, and an engineering agent share one model of context, one activity stream and one permission system. A user can start with a question on the bus, turn it into a plan, delegate the plan to an agent that clones the repository and runs the tests in an isolated cloud machine, watch it work, approve the one risky step, review the diff on a foldable, and open the pull request — without a laptop.

### 1.2 Product strategy

1. **Phone-first control surface, cloud-first execution.** All heavy work (clone, build, test, browse, index) happens in Xavro's cloud workbench. The Android app is the best possible place to direct, observe, review and approve. This is the gap nobody occupies.
2. **One workspace, one context model.** Conversations, research, files, repositories and missions all attach to Projects, so context accumulates instead of being re-explained.
3. **Trust as a feature.** Visible plans, a truthful activity trace, ground-truth approvals, budgets announced before work starts, and a written report at the end of every mission.
4. **Provider-agnostic intelligence.** Route each step to the right model; never let a provider's outage, pricing change or quality regression take the product down.
5. **Start lean, earn complexity.** A small number of well-chosen components (Postgres, a durable execution engine, managed microVM sandboxes, object storage) that each scale far before replacement is needed.

### 1.3 Target users

| Segment | Who | Why Xavro | Priority |
|---|---|---|---|
| **Mobile-heavy developers** | Indie developers, open-source maintainers, students, developers in regions where the phone is the primary computer | Real engineering work (fix, test, PR) without a laptop | Primary (launch) |
| **Professional developers away from desk** | Engineers on call, commuting, in meetings | Triage an issue, delegate a fix, review and approve from the phone, finish on desktop | Primary (launch) |
| **Technical leads / maintainers** | People who review more than they write | Delegate, review agent PRs, keep a project's context and conventions in one place | Secondary (launch) |
| **AI power users (non-developers)** | Analysts, writers, researchers, founders | Premium assistant with research reports, files and memory that stays organized by project | Secondary (launch, assistant features only) |
| **Small teams** | 2–20 person engineering teams | Shared projects, shared agent history, admin controls | Later (Professional phase) |
| **Enterprises** | Regulated orgs | SSO, audit, data residency, BYO cloud sandbox | Flagship phase |

### 1.4 User problems

1. "I see an issue on my phone but I can't do anything about it until I'm at my computer."
2. "My AI chats are scattered and forget everything about my project."
3. "Cloud agents give me a PR at the end, but I can't see or steer what happened in between."
4. "I don't trust an agent with my repository or my credentials."
5. "I don't know what a task will cost or why my quota ran out."
6. "Reviewing a diff on a phone is miserable."
7. "Approving agent actions is either constant interruptions or blind trust."
8. "Every model is best at something different and switching products is painful."

### 1.5 Jobs to be done

- **Ask:** "When I have a question, help me get a correct, sourced answer fast."
- **Research:** "When I need to understand a topic, investigate many sources and give me a report I can trust and reuse."
- **Understand:** "When I open a repository, explain how it works and where things are."
- **Delegate:** "When I know what I want changed, do it in a real environment, prove it works, and give me a reviewable result."
- **Steer:** "When the agent is working, let me see what it is doing and redirect it."
- **Review & approve:** "When the agent needs me, show me exactly what will happen and let me decide in one tap."
- **Remember:** "Keep my conventions, decisions and preferences so I never repeat them."
- **Resume:** "When I come back hours later, show me what happened and what needs me."

### 1.6 Core value

**Xavro turns a phone into a complete, safe command center for AI-driven engineering and research work.** The unit of value is a **verified outcome** (a passing pull request, a cited report, a working fix), not a chat message.

### 1.7 Product principles

1. **Outcome over output.** Every mission ends with a verified result or an honest explanation of why not.
2. **Show the work, not the thoughts.** Expose actions, observations, decisions and evidence; never expose private chain-of-thought.
3. **Ground truth at the point of decision.** Approvals display the real diff, the real command, the real recipient — rendered by trusted code, not summarized by the model.
4. **Enforcement lives on the server.** The UI explains permissions; trusted backend components enforce them.
5. **Context compounds.** Anything learned in a project (conventions, decisions, file map) is reusable by every future conversation and mission in that project, and is user-editable.
6. **Honest states only.** No fake data, fake success, fake progress or placeholder features presented as real. Unavailable means visibly unavailable, with the reason.
7. **Calm by default.** Notifications only when the user is needed or a mission finishes. No streaks, no engagement bait.
8. **Thumb-reachable power.** Every critical action on a phone is reachable one-handed; every dense view has a phone-appropriate form.
9. **Predictable cost.** Users see a budget before a mission starts and live spend while it runs.
10. **Replaceable parts.** Every external dependency sits behind an interface Xavro owns.

### 1.8 Differentiation

| Dimension | Status quo | Xavro |
|---|---|---|
| Surface | Desktop IDE/terminal; mobile is an afterthought | Native, adaptive Android control surface designed for agent work |
| Mid-run visibility | Final PR or opaque logs | Live **Trace** of actions, a **Trace Rail** progress signature, steerable at any time |
| Approvals | Per-tool prompts or none | Risk-classed **Checkpoints** with ground-truth previews, batched, with "remember for this mission" scoping |
| Context | Per chat or per repo | Per **Project**: conversations, research, missions, files and memory share one context |
| Cost | Hidden quotas | Budget estimate before start, live meter, hard caps |
| Models | One vendor or manual picker | Policy-based routing with capability registry and automatic fallback |
| Results | A diff | A **Mission Report**: what changed, why, evidence (tests, screenshots), risks, follow-ups |
| Security | Agent often holds tokens | Sandbox never holds long-lived credentials; brokered, scoped, expiring access |

### 1.9 Competitive positioning

Xavro does not compete with desktop IDE agents on "best place to type code". It competes on **"best place to direct and own AI work when you are not at your desk — and still very good when you are"**. Against chat assistants it wins on projects, execution and verified outcomes; against cloud agents it wins on visibility, steering, approvals and mobile review; against local agents it wins on safety and availability (no machine required).

Positioning statement: *For developers and builders who want real work done from anywhere, Xavro is the AI workspace that plans, builds and verifies in a secure cloud workbench, and puts you in control from your phone. Unlike desktop-bound agents, Xavro makes every step visible, reviewable and approvable on the device you always carry.*

### 1.10 North Star metric

**Weekly Verified Outcomes per active user (WVO):** the number of missions or research tasks per weekly active user that end in an accepted result — a merged or approved PR, an applied change, or a report the user saved or shared — with no rollback within 7 days.

Supporting metrics:

- Mission acceptance rate (accepted / completed missions).
- Steering-to-success ratio (missions that needed user steering but still succeeded).
- Median time-to-first-useful-signal (first meaningful Trace event after mission start).
- Checkpoint response time and checkpoint rejection rate (a high rejection rate means the agent is asking for the wrong things).
- Cost per verified outcome.
- 4-week retention of users with at least one verified outcome.

Counter-metrics (must not degrade): security incidents, unintended external actions, user-reported "agent did something I didn't expect", notification opt-out rate.

### 1.11 Product ecosystem

```
                         ┌───────────────────────────────┐
                         │   Xavro Android app (primary)  │
                         │  Ask · Projects · Missions ·   │
                         │  Library · Profile/Settings     │
                         └──────────────┬────────────────┘
                                        │ HTTPS (REST + SSE), WebSocket (terminal), FCM
┌──────────────────┐     ┌──────────────▼───────────────┐     ┌──────────────────────┐
│ Xavro Web Console │────▶│        Xavro Cloud           │◀────│  Xavro Developer API │
│ (admin, billing,  │     │ API · Agent runtime · Model  │     │  (Platform phase)     │
│  API keys, docs,  │     │ gateway · Workbench · Index  │     └──────────────────────┘
│  desktop review)  │     └──────┬───────────┬──────────┘
└──────────────────┘            │           │
                       ┌────────▼──┐   ┌────▼───────────────┐
                       │  GitHub   │   │ Model providers,   │
                       │  (App)    │   │ MCP integrations,  │
                       └───────────┘   │ web search          │
                                       └─────────────────────┘
```

- **Android app**: the flagship and the only end-user client at launch.
- **Web Console**: administration, billing, API keys, documentation and an optional desktop review view. Deliberately **not** a second full client at launch; it exists because admin, billing and key management do not belong on a phone and because reviewers sometimes want a big screen.
- **Xavro Cloud**: everything that thinks, executes or stores.
- **Developer API**: later phase; exposes missions, projects and the model gateway to other tools.
- **MCP**: Xavro is an MCP **client** from the MVP (connect approved tool servers) and becomes an MCP **server** in the Platform phase (let other agents read Xavro projects and start missions).

---

## 2. Technology discovery and selection

Each category below compares realistic candidates against the criteria in the brief (capability, maturity, security, ecosystem, cost, Android suitability, agent suitability, lock-in, future viability) and records a decision. "Port" means Xavro defines an internal interface so the choice can be replaced.

### 2.1 Android client stack

| Option | Assessment |
|---|---|
| **Kotlin + Jetpack Compose (native)** | STABLE. Best adaptive-layout support (Material 3 Adaptive, Navigation 3), best performance for long streaming lists and terminals, full platform integration (Credential Manager, FCM, WorkManager, share sheet, foldable postures). |
| Flutter | STABLE. Good cross-platform reach; adaptive Android-specific behaviour (pane scaffolds, posture, predictive back, desktop windowing) lags native and must be rebuilt. |
| React Native | STABLE. Fast for web teams; streaming-heavy views and large diffs need native modules; feels least native. |
| Kotlin Multiplatform + Compose Multiplatform | STABLE for shared logic, Compose Multiplatform on iOS STABLE since 2025. Adds build complexity before there is an iOS plan. |

**Decision:** Native **Kotlin + Jetpack Compose**, Material 3 as the component foundation with a custom Xavro token layer, **Compose Material 3 Adaptive 1.3** and **Navigation 3**. Domain and protocol modules are **pure Kotlin (KMP-ready)** so an iOS or desktop client can later share them without a rewrite. **Why:** the product's differentiator is a superb Android experience; native is the only way to get adaptive panes, posture awareness and smooth streaming without compromise.

Supporting libraries (all STABLE unless noted): Hilt (DI), kotlinx.coroutines/Flow, kotlinx.serialization, OkHttp (HTTP/2, SSE via okhttp-sse, WebSocket), Room (offline cache), DataStore (preferences), WorkManager (deferred sync/uploads), Coil (images), Credential Manager (passkeys + Sign in with Google), Firebase Cloud Messaging (push), Baseline Profiles + Macrobenchmark (startup and jank), JetBrains `markdown` parser (Kotlin, multiplatform) rendered to Compose.

**Code viewing and editing:**

| Option | Assessment |
|---|---|
| Native Compose viewer with server-supplied syntax tokens | Best performance and theming; read-only and diff views only. |
| CodeMirror 6 in an isolated WebView | Mature editor (STABLE), full editing features; a WebView inside a native app, so it must be contained. |
| Native Android editor libraries (e.g. Sora Editor) | Capable; licence and maintenance must be checked before adoption. |

**Decision:** All **viewing and diff review** is native Compose using highlight tokens computed server-side with tree-sitter (one highlighting source for app and web). **Editing** (short edits, conflict resolution) uses a **locked-down CodeMirror 6 WebView component** (no network, local assets only, message-port bridge) behind a `CodeEditor` port; re-evaluate a native editor in the Professional phase. Rationale: on a phone, 90% of code interaction is reading and reviewing.

**Terminal rendering:** a native Compose VT/ANSI renderer backed by a proven terminal-emulator state machine (evaluate the Apache-2.0 Termux `terminal-emulator` library; verify licence) behind a `TerminalSurface` port. The PTY always runs in the cloud workbench, never on the device.

### 2.2 Backend language and framework

| Option | Assessment |
|---|---|
| **Kotlin on JVM 21 (Ktor)** | STABLE. Shares protocol/domain code with Android via KMP; virtual threads + coroutines for heavy streaming; official SDKs exist for Anthropic, OpenAI and Google GenAI (Java/Kotlin), official **MCP Kotlin SDK**, **Restate Kotlin SDK**, Temporal Java SDK. One language across client and server. |
| TypeScript (Node/Bun) | STABLE. Largest AI-agent library ecosystem; weaker type sharing with Android; runtime footguns for long-running CPU work. |
| Python | STABLE. Best ML ecosystem; Xavro does not train models; concurrency and packaging overhead for a streaming API. |
| Go | STABLE. Excellent for infrastructure daemons; thinner AI SDK ecosystem; no code sharing with Android. |

**Decision:** **Kotlin (JVM 21, Ktor server)** for the API, agent runtime, model gateway and indexing coordinator, with a shared **`xavro-protocol`** KMP module (event schemas, API DTOs, tool schemas) used by Android and server. **Go** is reserved for the self-hosted workbench supervisor in a later phase (Firecracker's tooling is Go-native). **Why:** one language and one schema definition across the most important boundary (app ↔ server) eliminates a whole class of drift bugs, and every critical SDK is available.

### 2.3 Agent framework

| Option | Assessment |
|---|---|
| LangGraph / LangChain | STABLE, popular; Python/TS; heavy abstractions; durability is its own concern. |
| Vendor agent SDKs (OpenAI Agents SDK, Claude Agent SDK, Google ADK) | STABLE; each optimizes for its vendor's models and conventions. |
| **Own minimal agent loop on a durable execution engine** | Full control over permissions, events, budgets, routing and durability; small (the loop itself is a few hundred lines; the value is in tools and context). |

**Decision:** **Build the agent loop in-house** on top of the durable engine (§2.5) and the model gateway (§2.4). Use **MCP** as the tool-integration standard. **Why:** permission enforcement, event emission, budget control and provider neutrality are Xavro's core product; they cannot sit inside a third-party abstraction. Frameworks remain useful as reference designs.

### 2.4 Model access layer

| Option | Assessment |
|---|---|
| Direct vendor SDKs everywhere | Maximum fidelity; provider logic leaks into business code. |
| LiteLLM / Portkey (gateway products) | STABLE; fast to start; normalization loses vendor-specific features (prompt caching controls, reasoning settings, server tools). |
| OpenRouter | STABLE; huge catalogue; extra hop, data-handling terms per downstream provider. |
| **In-house Model Gateway with provider adapters** | Exact control of streaming, tool calls, caching, cost attribution, fallbacks and data policy. |

**Decision:** **In-house `ModelGateway`** with first-class adapters for **Anthropic, OpenAI and Google** (official SDKs or raw HTTP), an **OpenAI-compatible adapter** for open-weight hosts, and an optional **OpenRouter adapter** for the long tail. Normalized internal message/tool/stream model, per-model capability registry, routing policies and fallbacks (§15).

### 2.5 Durable execution (missions)

| Option | Fit for Xavro |
|---|---|
| Temporal (self-host or Cloud) | Most proven. Heavier to operate self-hosted; per-action pricing on Cloud grows with chatty agent loops. |
| **Restate** | Single binary, Kotlin SDK, Virtual Objects map exactly to "one mission = one keyed object with serialized state", awakeables model "wait for user approval" with no polling, BYOC available. Younger. |
| DBOS | Postgres-only and very lean; ecosystem and language coverage narrower for Kotlin. |
| Home-grown on Postgres | Reinvents retries, timers, idempotency, replay. |

**Decision:** **Restate** behind an internal **`MissionEngine` port**, with **Temporal as the pre-evaluated fallback**. **Why:** missions are long-lived keyed state machines that pause for humans; Restate's model fits that shape with the least infrastructure. **Risk mitigation:** the port confines engine APIs to one module, and an engine-swap spike is scheduled before the MVP freezes (§30).

### 2.6 Execution sandbox (the Workbench)

| Option | Fit |
|---|---|
| **Managed microVM sandbox provider** (E2B, Daytona, Modal, others) | Fastest path to strong isolation; snapshot/resume, per-second billing; vendor dependency. |
| Self-hosted Firecracker (+ jailer) on bare metal / nested-virt VMs | Strongest control and lowest marginal cost at scale; significant ops work. |
| gVisor on Kubernetes | Easy to run anywhere; I/O-heavy agent loops pay syscall overhead. |
| Plain containers | **Rejected**: shared kernel is not an acceptable boundary for untrusted, agent-generated code. |

**Decision:** **MVP uses a managed microVM-class provider** (selection criteria: hardware-virtualization isolation, snapshot/pause-resume, egress control hooks, region choice, SOC 2, API for PTY streaming) behind a **`WorkbenchProvider` port**. **Professional phase** adds a **self-hosted Firecracker pool** (Go supervisor, jailer, snapshot-based warm pool) for cost and data-residency, keeping the managed provider as overflow. gVisor is the fallback only for environments without KVM.

### 2.7 Data stores

| Need | Decision | Alternatives considered |
|---|---|---|
| System of record | **PostgreSQL** (managed, latest major supported by provider), STABLE | MySQL, CockroachDB (not needed at start), document DBs (weaker relational integrity) |
| Vectors | **pgvector (HNSW)** in a separate Postgres database from the OLTP primary | Qdrant, Turbopuffer (adopt when vectors exceed ~50M or index rebuilds hurt OLTP) |
| Lexical search over indexed code/docs | **Postgres full-text** for metadata; **ripgrep inside the workbench** for live trees | Elasticsearch/OpenSearch (operational cost not justified early) |
| Blobs (attachments, artifacts, logs, snapshots, index files) | **S3-compatible object storage** | — |
| Cache, rate limits, ephemeral fan-out | **Valkey** (open-source Redis fork) | Redis (licence changes), Memcached (no streams) |
| Event fan-out | Persist events in **Postgres** (`run_events`, ordered by sequence) + live fan-out over **Valkey pub/sub**; move to **NATS JetStream** when fan-out or replay volume demands | Kafka (too heavy early) |
| Analytics warehouse | **ClickHouse** (later phase) fed by event exports | BigQuery/Snowflake (cloud lock-in) |

### 2.8 Identity and authentication

| Option | Assessment |
|---|---|
| Auth0 / Okta CIC | STABLE, full-featured, expensive at scale, lock-in. |
| WorkOS AuthKit | STABLE, strong enterprise SSO story. |
| Firebase Auth | STABLE, Android-friendly, thin enterprise features. |
| **Zitadel** (open source; cloud or self-host) | STABLE; OIDC, passkeys/WebAuthn, social login, organizations, SSO federation; no lock-in. |
| Keycloak | STABLE; heavy to operate. |

**Decision:** **OIDC-standard identity via Zitadel (managed cloud at launch, self-host option later)**; Android signs in with **Credential Manager** (passkeys first, Sign in with Google second, email magic link fallback). Xavro's API issues its **own short-lived access tokens** (≤15 min) and rotating refresh tokens bound to the device, so the identity vendor is swappable. GitHub is a **linked account**, never the primary identity.

### 2.9 Code intelligence

**Decision:** **tree-sitter** for parsing, chunking, outlines and highlighting; **SCIP indexers** (when the project builds) for precise cross-references; **ripgrep** in the workbench for live search; **embeddings** (provider via gateway; default a code-tuned embedding model) for semantic ranking; hybrid retrieval with reciprocal-rank fusion. Indexes are keyed by repository + commit SHA and built incrementally inside the workbench (§10).

### 2.10 Browser automation

| Option | Assessment |
|---|---|
| **Playwright (Chromium) inside the workbench** | STABLE; accessibility snapshots, screenshots, tracing, network control; also used for the user's own app verification. |
| Puppeteer | STABLE; Chromium-only, smaller tooling. |
| Hosted browser APIs | STABLE-PREVIEW; useful for scale; data leaves Xavro's boundary. |

**Decision:** **Playwright + headless Chromium running inside the mission's own workbench VM** (so it can reach the app under test on localhost and nothing else unless egress allows). Exposed to the agent as Xavro browser tools (accessibility snapshot first, screenshot second). Agentic browsing of the open web for research uses a separate, more restricted **read-only fetch/browse tool** (§13, §16).

### 2.11 Observability, analytics, errors

**Decision:** **OpenTelemetry** SDKs everywhere (Ktor server, workers, Android via the OTel Android SDK for network/span correlation — PREVIEW, wrapped). Backends: **Grafana stack** (Tempo traces, Loki logs, Mimir/Prometheus metrics) self-hosted or Grafana Cloud; **Langfuse** (self-hosted) for LLM/agent traces and eval datasets, fed from the same OTel spans using GenAI semantic conventions (pinned version, development status). **Sentry** for Android crash/ANR and server exceptions. **PostHog** (self-hosted or EU cloud) for product analytics with strict event allowlists (no prompt content).

### 2.12 Cloud and delivery

**Decision:** **Google Cloud** for launch (managed Postgres via Cloud SQL or AlloyDB, Cloud Run for stateless services, GKE Autopilot for workers that need it, Cloud Storage, Secret Manager, Cloud KMS, FCM synergy, Gemini via Vertex as one of several providers), with **portability rules**: everything containerized, infrastructure as code in **OpenTofu**, no proprietary database features, S3-compatible storage abstraction. **CI/CD:** GitHub Actions (STABLE) with OIDC federation to cloud (no long-lived deploy keys), signed container images, SBOMs, Play App Signing, Gradle managed devices for UI tests.

**Rejected as a launch default:** Kubernetes-everything (premature), multi-cloud active/active (premature), serverless functions for the agent loop (execution time limits, cold starts).

### 2.13 Policy and authorization

**Decision:** Authorization is enforced in one server module, **`Policy`**, implemented as typed Kotlin rules with exhaustive tests at launch; adopt **Cedar** (open-source policy language with formal analysis, STABLE) when organization admins need custom policies (Professional phase). Every tool call, API call and approval passes through `Policy.decide(principal, action, resource, context)`.

### 2.14 Notifications

**Decision:** **FCM** for push (data messages, high priority only for checkpoints and mission completion). Android notification channels per category so users control them in system settings. Email (via a transactional provider behind a port) only for account/security events and opt-in digests.

### 2.15 Summary of build vs integrate

| Build in-house (core IP) | Integrate (commodity) |
|---|---|
| Agent loop, planner, verifier | Model providers |
| Model gateway & routing | Durable execution engine (Restate) |
| Tool system & permission policy | Sandboxes (managed microVMs → own Firecracker later) |
| Project intelligence (index orchestration, retrieval, repo map) | tree-sitter, SCIP indexers, ripgrep |
| Event system & Trace | Postgres, Valkey, object storage |
| Android app & design system | Identity provider (Zitadel), FCM |
| Mission Report, Checkpoint UX | Observability backends, Sentry, PostHog |
| Credential broker / egress proxy | Web search API(s) |

---

## 3. Product structure and information architecture

### 3.1 What exists, where it lives, and what does not exist

| Area | Placement | Reason |
|---|---|---|
| **Ask** (conversations) | Primary destination | The universal entry point; every interaction can start here. |
| **Projects** | Primary destination | The container for context: repository, files, conversations, missions, memory. |
| **Missions** (agent runs) | Primary destination | Delegated work is a first-class object; users return to it to steer, approve and review. Shows a badge when a checkpoint waits. |
| **Library** (artifacts and files across projects) | Primary destination | Reports, diffs, generated files, uploads, screenshots in one place. |
| **Profile & Settings** | Avatar entry in the top bar | Not a frequent destination. |
| Repository / Code browser | Inside a Project | Repositories only make sense in project context. |
| GitHub | Inside a Project (repo, branches, PRs, issues) + Settings (account link, installations) | Not a separate world. |
| Files | Inside a Project + Library | Same files, two lenses. |
| Terminal | Inside a Project's **Workbench** and inside a Mission | It belongs to a running environment, not to the app. |
| Agent activity (Trace) | Inside a Mission; summarized in Ask when a mission is started from chat | Activity without its mission is noise. |
| Research | A **mode of Ask** that produces a Report artifact | Not a separate app; research is a kind of conversation with a heavier engine. |
| Memory | Project → Memory tab; Settings → Personal memory | Users must see and edit memory where it applies. |
| Models | Composer model chip; Settings → Models (defaults, routing preference, BYOK later) | Selection is contextual; configuration is rare. |
| Tools & integrations | Settings → Integrations; Project → Tools (enabled per project) | Tools are permissions; they belong to configuration. |
| History | Search across Ask, Missions, Library (global search) | No separate "History" destination; search is history. |
| Notifications | System notifications + an **Inbox** sheet from the top bar (checkpoints, completions, mentions) | Avoid a fifth tab. |
| Usage & billing | Settings → Usage (view); Web Console → Billing (manage) | |
| Security | Settings → Security (devices, sessions, passkeys, connected accounts, audit of agent actions) | |
| API & Documentation | Web Console only | Not a phone job. |
| Administration | Web Console only | Not a phone job. |
| "Tasks" (to-do lists) | **Does not exist** as a separate area | Missions already are tasks; a second task system confuses. |
| "Agent marketplace" | **Does not exist** at launch | Supply-chain risk (ASI04) and no user evidence. FUTURE. |
| Social/feeds | **Does not exist** | Not aligned with calm principle. |

### 3.2 Navigation model

- **Phone (compact width):** bottom navigation bar with four destinations: **Ask · Projects · Missions · Library**. Top app bar: global search, Inbox (badge), avatar.
- **Medium width (foldable inner display portrait, small tablet):** navigation rail with the same four destinations + a compose FAB at the top of the rail.
- **Expanded / large / extra-large (tablet landscape, desktop windowing):** navigation rail (or permanent drawer on extra-large) + list-detail panes + optional supporting pane (Trace, Files, Terminal).

Implemented with `NavigationSuiteScaffold` (wrapped behind Xavro's own `XNavigationSuite` because its override APIs are changing) and Navigation 3 back stacks with adaptive scene strategies.

### 3.3 Core object model (user-facing)

```
User
 ├─ Personal memory
 ├─ Conversations (unfiled)            ← Ask
 └─ Projects
     ├─ Overview (status, recent activity, open checkpoints)
     ├─ Conversations (project-scoped)
     ├─ Missions
     │    ├─ Plan
     │    ├─ Trace (live activity)
     │    ├─ Checkpoints
     │    ├─ Changes (diff)
     │    ├─ Terminal / Browser / Files (live)
     │    └─ Mission Report
     ├─ Repository (one primary repo at launch; multi-repo later)
     │    ├─ Code browser · Branches · Pull requests · Issues
     ├─ Files (uploads + generated)
     ├─ Memory (project instructions, decisions, conventions)
     ├─ Workbench (environment definition: image, setup, secrets refs, egress allowlist)
     └─ Settings (tools enabled, models, budgets, permissions)
```

### 3.4 Terminology (Xavro's own vocabulary)

| Xavro term | Meaning | Why this word |
|---|---|---|
| **Ask** | Conversational surface | A verb; invites action rather than "chat". |
| **Project** | Context container | Plain and universally understood. |
| **Mission** | A delegated, multi-step job executed by the agent | Signals autonomy with a defined objective and an end. |
| **Plan** | The mission's ordered steps, editable before and during execution | — |
| **Trace** | The truthful, user-safe activity stream of a mission | Implies evidence, not thoughts. |
| **Trace Rail** | The compact live progress signature shown everywhere a mission appears | Xavro's signature UI element. |
| **Checkpoint** | A point where the mission needs the user's decision | Neutral, not alarming; implies safe resumption. |
| **Workbench** | The isolated cloud machine where a project's code runs | Physical metaphor; a place where work happens. |
| **Mission Report** | The structured outcome document | Every mission ends with one. |
| **Library** | Artifacts and files across everything | — |
| **Memory** | What Xavro remembers, always visible and editable | Plain; avoids anthropomorphic terms. |
| **Signals** (internal) | Real-time events | Not shown to users. |

---

## 4. Complete UX plan

### 4.1 Journey map overview

```
Install → Launch → Welcome → Sign in (passkey/Google) → Personalize (2 questions) →
First Ask → (optional) Create Project → Connect GitHub (App install, choose repos) →
Workbench prepares (index + environment detection) → First Mission (guided) →
Checkpoint → Review Changes → Verification evidence → Open PR → Mission Report →
Return visits (Inbox, Missions, Projects) → Collaboration (Professional phase)
```

### 4.2 Journeys in detail

Each journey lists goal, entry, interaction, system behaviour, feedback, and the required loading / empty / failure / permission / recovery / completion states.

#### J1. Install and first launch

- **Goal:** understand what Xavro is and get to a first answer in under 60 seconds.
- **Entry:** Play Store install, app icon.
- **Interaction:** splash (system splash API, no custom delay) → one Welcome screen with three short statements (Ask anything · Delegate real work · Stay in control) and two buttons: **Continue with passkey/Google**, **Other sign-in options**.
- **System:** checks Play Integrity verdict in the background (used only for abuse scoring, never to block legitimate users silently); pre-fetches public model list.
- **Feedback:** immediate; no carousel.
- **Loading:** none beyond system splash.
- **Failure:** offline → Welcome still renders; sign-in buttons show "You're offline" inline with a retry.
- **Completion:** signed-in user lands in Ask with the composer focused.

#### J2. Authentication

- **Goal:** secure sign-in without passwords.
- **Interaction:** Credential Manager bottom sheet. New users: create passkey (or Sign in with Google). Fallback: email magic link (deep link back into the app).
- **System:** OIDC with PKCE against Zitadel; Xavro API exchanges the ID token for Xavro session tokens; device registered (device id, public key for token binding, FCM token).
- **Permission state:** notification permission is **not** requested here; it is requested contextually the first time a mission starts (J6).
- **Failure:** passkey cancelled → stays on sheet with alternative options; provider error → message with retry and "Use email instead"; account locked → support path.
- **Recovery:** lost device → sign in on new device with Google/email, revoke old device in Settings → Security.
- **Completion:** session established; refresh token stored in Android Keystore-backed encrypted storage.

#### J3. Personalization (optional, skippable)

- Two questions maximum: "What will you mostly use Xavro for?" (Coding · Research & writing · Both) and "Preferred response style" (Concise · Detailed). Answers seed personal memory as user-visible entries that can be edited later.

#### J4. First conversation (Ask)

- **Goal:** a fast, high-quality answer.
- **Interaction:** composer at bottom; model chip shows "Auto" (routing). Suggestions shown only on an empty Ask: four contextual starters, two of which demonstrate projects/missions.
- **System:** request → API → ModelGateway (routing) → streamed SSE tokens; Trace-lite for tool use (web search, file read) shown as inline activity chips.
- **Loading:** a typing indicator replaced within ~300 ms by streaming text; if the first token takes longer than 2 s, show "Thinking" with the selected model name; after 10 s show elapsed time and a Stop button remains.
- **Empty:** suggestions, plus "Start a project" card.
- **Failure:** provider error → automatic fallback to the next model in policy with a small "Answered by X (fallback)" note; full failure → message bubble with error reason and **Retry** / **Try another model**. Partial stream preserved.
- **Cancellation:** Stop keeps partial output marked "Stopped".
- **Completion:** answer with actions: copy, share, regenerate, branch, save to project, turn into mission (if actionable).

#### J5. First project and connecting a repository

- **Goal:** give Xavro durable context and a repository.
- **Entry:** Projects → New project; or from an Ask answer "Save to project"; or "Start a project" card.
- **Interaction:** name + optional description → "Add a repository" (GitHub) or "Skip" (non-code projects for research/files).
- **GitHub flow:** if not linked → **Connect GitHub** opens a Custom Tab to install the Xavro GitHub App, user selects repositories on GitHub's own page → redirect back via App Link → Xavro lists the installation's repositories → user picks one.
- **System:** webhook confirms installation; Xavro creates the project-repo binding; schedules **Workbench preparation**: clone at default branch in a fresh workbench, detect languages/build systems/test commands, build tree-sitter + (where possible) SCIP index, compute embeddings, generate a **Repo Map** (summary of structure, entry points, test commands), read existing instruction files (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `.github/copilot-instructions.md`, `README`) as project-memory candidates.
- **Feedback:** a preparation card with real steps (Cloning · Detecting stack · Indexing 1,240 files · Summarizing) and a Trace Rail.
- **Loading:** project usable immediately for chat; code-aware features show "Indexing…" with progress.
- **Empty:** project with no repo shows Files and Conversations prominently.
- **Failure states:** app install not completed → "GitHub didn't finish connecting" with Retry; repo too large for plan → explicit limit message and option to index a subdirectory; clone failure (private submodule, LFS quota) → specific error with fix suggestions; detection uncertain → asks the user to confirm the test command.
- **Permission state:** shows exactly which permissions the GitHub App holds and on which repos (read from GitHub, not assumed).
- **Completion:** Project Overview shows Repo Map summary, detected commands, suggested first missions derived from open issues.

#### J6. First mission

- **Goal:** delegate a real change and get a verified result.
- **Entry:** Project → "New mission"; Ask message that is actionable → "Run as mission"; GitHub issue → "Start mission from issue"; Android share sheet (share a GitHub issue URL into Xavro).
- **Interaction (Mission Composer):** objective (text/voice/attachments), project + repo + base branch (prefilled), **mode** (Plan first [default for new users] · Go), **budget** (estimated cost range and time range from historical data for similar missions; user can set a cap), **autonomy level** (Careful · Balanced [default] · Autonomous; see §19).
- **System:** intent analysis → if underspecified, the agent asks up to three **clarifying questions** as a structured card before planning → context collection → **Plan** produced → in Plan-first mode, the user reviews/edits/approves the plan (this is the first Checkpoint) → execution.
- **Contextual permission:** first mission requests the Android notification permission with a rationale screen ("We'll notify you only when a mission needs you or finishes").
- **Feedback:** Mission screen with Trace Rail at top, Plan with step states, Trace stream, tabs for Changes / Terminal / Browser / Files.
- **Loading:** workbench resume from snapshot (target < 5 s) shows "Preparing workbench" step in the Trace.
- **Failure:** see §8 recovery rules; every failure becomes a Trace event and, when unrecoverable, a Mission Report with status **Blocked** or **Failed** and the specific reason.
- **Completion:** Mission Report + "Open pull request" (if not already opened per autonomy settings).

#### J7. Checkpoint (approval)

- **Goal:** the user decides quickly and correctly.
- **Entry:** push notification (actionable when safe: Approve/Decline buttons only for R2 risk; R3+ always opens the app), Inbox, Mission screen banner.
- **Interaction:** Checkpoint sheet shows: **what** (action type in plain words), **exact payload** rendered by trusted code (diff, command line, URL, PR title/body, recipient), **why** (the agent's stated reason, visually separated and labelled as the agent's explanation), **risk class** and consequences, and choices: **Approve**, **Approve and allow similar for this mission**, **Edit** (where meaningful: edit command, PR body), **Decline with note**.
- **System:** decision recorded with user, device, time, and payload hash; mission awakened; policy cache updated for "allow similar".
- **Failure:** expired checkpoint (default 24 h, configurable) → mission pauses in **Waiting** state; user can resume later.
- **Completion:** sheet dismisses with haptic confirmation; Trace shows the decision.

#### J8. Reviewing changes

- **Goal:** understand and trust a diff on a phone.
- **Interaction:** **Changes** tab: file list grouped by significance (source / tests / config / generated), each file with +/- counts and an agent-written one-line rationale; tap → native diff view (unified on compact, split on expanded), syntax-highlighted; long-press a line → comment to the agent ("change this", "why?") which becomes steering input; "Explain this file's changes" asks a fast model with the diff as context.
- **Evidence:** test results attached (pass/fail with log excerpts), screenshots for UI changes, lint/type-check results.
- **Failure:** very large diffs → summary first, virtualized lists, generated files collapsed.
- **Completion:** "Looks good → Open PR" or "Request changes" (steering message).

#### J9. Verification

- The agent must show **evidence**: commands run with exit codes, test summary, before/after screenshots for UI, and a self-review checklist. The Mission Report marks each claim as **Verified** (backed by a tool observation) or **Unverified** (agent assertion).

#### J10. Opening a PR and collaboration

- PR created by the Xavro GitHub App on a `xavro/<mission-slug>` branch, as **draft** by default, body generated from the Mission Report, with the user credited as requester. Review comments on the PR flow back as mission events; the user can let the agent address them (new mission iteration on the same branch).
- **Collaboration (Professional phase):** shared projects, mission handoff, comments on mission steps, team inbox.

#### J11. Continued usage (returning user)

- Opening the app shows **Ask** by default, but a **"Needs you" strip** appears at the top whenever any checkpoint is waiting or a mission finished since the last visit. The Missions tab shows running missions with live Trace Rails.
- Long-absence return (>24 h): a digest card summarizes what finished, what's blocked, and what's waiting — once, dismissible.

### 4.3 Friction removed

| Friction | Removal |
|---|---|
| Setting up a dev environment on a phone | Workbench auto-detects stack and test commands; user confirms instead of configuring. |
| Explaining the project every time | Project memory + Repo Map attached automatically. |
| Approval fatigue | Risk classes, batched checkpoints, "allow similar for this mission", autonomy levels. |
| Losing track of a mission | Trace Rail everywhere, Inbox, push only when needed. |
| Unclear cost | Budget estimate before start, live meter, hard cap. |
| Reading huge diffs on small screens | Grouping, rationales, explain-this, evidence-first review. |
| Re-authenticating GitHub | GitHub App installation persists; short-lived tokens minted server-side. |

---

## 5. Visual design language: "Meridian"

### 5.1 Identity

Meridian is Xavro's design language. A meridian is a line of reference used for navigation — calm, precise, and oriented. The language expresses **instrument-grade clarity**: the feel of a well-made tool (a precision instrument, a cockpit at night, a technical drawing), not a neon "AI" dashboard.

Adjectives translated into rules:

| Quality | Concrete rule |
|---|---|
| Premium | Few elements, generous spacing, high-quality type, no stock gradients, 1 dp hairlines instead of heavy shadows, consistent optical alignment. |
| Futuristic, not childish | Geometric precision, monospaced data, the Trace Rail signature; no mascots, sparkles, or glowing orbs. |
| Calm | Neutral surfaces dominate (≥85% of any screen); colour reserved for state and action; motion only to explain change. |
| Intelligent | Information density adapts to width; data is aligned and tabular; numbers use tabular figures. |
| Trustworthy | Risk is always colour + icon + text; agent-authored text is visually distinguished from system facts. |
| Developer-friendly | Monospace where content is code or data; copy affordances everywhere; real paths, real commands, real exit codes. |

### 5.2 Colour

Tokens are semantic; raw palette values are only referenced by tokens. Both themes meet WCAG 2.2 AA (4.5:1 body text, 3:1 large text and UI components); critical states meet AAA for text.

**Neutral ramp — "Graphite" (warm-neutral grey)**

| Token | Light | Dark |
|---|---|---|
| `surface.base` | #FAFAF8 | #0F1012 |
| `surface.raised` | #FFFFFF | #16171A |
| `surface.sunken` | #F1F1EE | #0B0C0D |
| `surface.overlay` | #FFFFFF (with scrim) | #1C1D21 |
| `outline.hairline` | #E3E2DE | #2A2B30 |
| `outline.strong` | #C9C8C2 | #3A3B42 |
| `text.primary` | #17181A | #ECECEA |
| `text.secondary` | #55565B | #A7A8AD |
| `text.tertiary` | #7C7D82 | #7D7E85 |

**Accent — "Verdigris"** (the brand colour; a mineral blue-green, chosen to avoid the purple/blue "AI" cliché while reading as technical and calm)

| Token | Light | Dark |
|---|---|---|
| `accent.primary` | #1F6F66 | #5CC2B3 |
| `accent.onPrimary` | #FFFFFF | #00201C |
| `accent.container` | #D3EEE9 | #123A35 |

**State colours** (always paired with icon + label)

| Token | Use | Light | Dark |
|---|---|---|---|
| `state.active` | Agent working, live | #1F6F66 (accent) | #5CC2B3 |
| `state.attention` | Checkpoint waiting, needs user | #B26A00 (amber) | #F2B35B |
| `state.success` | Verified, passed | #2E7D32 | #7BC67E |
| `state.danger` | Destructive, failed | #B3261E | #F2B8B5 |
| `state.info` | Neutral notice | #3B5BA9 | #A9BFF2 |
| `diff.added.bg/fg` | Diff | #E6F4EA / #1B5E20 | #13261A / #9FD8A4 |
| `diff.removed.bg/fg` | Diff | #FCE8E6 / #8C1D18 | #2C1513 / #F2B8B5 |

**Dynamic colour:** Material You dynamic colour is **offered as an opt-in setting** ("Match my wallpaper") that replaces only the accent family; state colours never change (safety semantics must be stable).

### 5.3 Typography

- **UI:** *IBM Plex Sans* (SIL OFL) — technical, humanist, excellent at small sizes, distinctive without being decorative.
- **Code, data, terminal, numbers in tables:** *IBM Plex Mono* (SIL OFL).
- **Arabic and other scripts:** *IBM Plex Sans Arabic* and Noto fallbacks; full RTL support is a launch requirement (the founding team's region makes Arabic a first-class language).
- Scale (sp): Display 34/40, Headline 24/30, Title 18/24, Body 15/22, Body-small 13/18, Label 12/16, Code 13/20 (phone) and 14/21 (expanded). Respect user font scaling up to 200% with reflow, never truncation of critical content.
- Tabular figures for all costs, durations, counts.

### 5.4 Iconography

- Outline icons on a 24 dp grid, 1.75 dp stroke, rounded joins (Material Symbols Rounded as the base set with weight 300, plus ~40 custom Xavro icons for Mission, Checkpoint, Workbench, Trace, Plan, Verified, Unverified, Repo Map).
- Filled variants only for selected navigation state and for state indicators.

### 5.5 Shape language

- Containers: 16 dp corner radius (cards, sheets top corners 28 dp per Material bottom sheet), chips 8 dp, inputs 12 dp, code blocks 10 dp.
- **Signature motif — the Meridian notch:** mission-related containers (Mission cards, Checkpoint sheets, Mission Report header) have a 2 dp accent line along the leading edge that carries the **Trace Rail** state colour. Instantly identifies "agent work" versus conversation.
- Elevation through tone (surface.raised vs base) and 1 dp hairlines; shadows only for floating elements (FAB, menus, sheets).

### 5.6 Spacing and layout grid

- 4 dp base unit; component padding 12/16 dp; screen gutters 16 dp (compact), 24 dp (medium), 32 dp (expanded+).
- Reading width capped at ~680 dp for prose in Ask; code and diffs may extend to pane width.
- Touch targets ≥ 48×48 dp.

### 5.7 Motion

- **Purpose-only motion:** explain state change, maintain spatial continuity, confirm actions. No idle animations except the live Trace Rail.
- Material 3 motion scheme (spring-based, "standard" for most, "expressive" only for mission completion).
- Durations: 150 ms micro (chip state), 250 ms component (expand step), 350 ms container transform (card → Mission screen), shared element between Mission card and Mission header.
- Streaming text appears by token without per-token animation (no "typewriter" fade); a caret indicates live output.
- **Reduced motion:** honour the system "Remove animations" setting; the Trace Rail becomes a static segmented bar.

### 5.8 The Trace Rail (signature element)

A thin horizontal rail (4 dp tall in cards, 6 dp in the Mission header) divided into segments for plan steps:

- Completed segments: solid accent.
- Current segment: animated progress shimmer moving at a slow, constant speed (not tied to fake percentages).
- Waiting on checkpoint: segment turns amber and pulses once per 2 s.
- Failed segment: danger colour with a notch.
- Pending: hairline outline.

The rail never fakes progress: segment widths are equal per step (not time-estimated), and the shimmer only means "this step is running now".

### 5.9 Feedback and micro-interactions

- Haptics: light tick on send, confirm on approve, heavy (single) on destructive confirmation, none during streaming.
- Copy actions show a 1.5 s inline "Copied" label, no toasts.
- Snackbars only for undoable actions ("Message deleted · Undo").
- Pull-to-refresh only on lists backed by server state (Missions, PRs); Ask streams do not need it.

### 5.10 Component behaviour (selected)

| Component | Behaviour |
|---|---|
| Composer | Grows to 6 lines then scrolls; attachments as chips above input; model chip and mode chip (Ask / Research / Mission) on the leading edge; voice button morphs into send when text exists. |
| Message (user) | Right-aligned (left in RTL), tonal container, editable. |
| Message (assistant) | Full-width, no bubble, prose typography, code blocks with copy/expand/"open in Workbench". |
| Activity chip | Inline compact row: icon + verb + object + duration ("Searched web · 'gradle 9 release' · 1.2 s"); tap expands to show inputs/outputs. |
| Mission card | Meridian notch, title, project, Trace Rail, status label, cost so far, "Needs you" badge. |
| Checkpoint sheet | Modal bottom sheet (compact) / dialog-pane (expanded); never dismissible by outside tap when risk ≥ R3. |
| Diff view | Unified (compact) / split (≥ 840 dp); line wrapping toggle; collapsed unchanged hunks; jump to next change via volume keys optional. |
| Terminal | Monospace, ANSI 256/truecolor, selectable text, extra-keys row (Esc, Tab, Ctrl, arrows, |, ~) above the keyboard, pinch to zoom font. |

### 5.11 Dark and light mode

Dark mode is designed first (most developer usage, OLED battery), light mode is equal-quality. Pure black is avoided for surfaces (#0F1012) to prevent smearing on OLED during scroll; an optional "True black" toggle exists for battery-focused users.

### 5.12 Accessibility

- TalkBack: every element labelled; Trace events announced politely (not assertively) and throttled to one announcement per 5 s; Checkpoints announced assertively.
- Streaming answers: screen reader reads completed paragraphs, not token-by-token.
- Colour is never the only carrier of meaning.
- Font scaling to 200%, display size, high-contrast text, switch access and keyboard navigation (tablets and Chromebooks/desktop windowing) supported.
- Focus order defined for every pane layout.
- RTL mirrored layouts, with code, diffs and terminal kept LTR.

---

## 6. Android experience

### 6.1 Window-size behaviour

Breakpoints follow Material window size classes (width): compact < 600 dp, medium 600–839, expanded 840–1199, large 1200–1599, extra-large ≥ 1600.

| Screen | Compact (phone) | Medium (foldable inner portrait) | Expanded+ (tablet landscape, unfolded landscape, desktop window) |
|---|---|---|---|
| Ask | Single pane conversation; history via top-bar drawer | Rail + conversation; history as list-detail | List (conversations) + detail (conversation) + optional supporting pane (sources/artifacts) |
| Projects | List → Project screen with tabs | List-detail | List-detail; Project detail itself uses tabs + supporting pane |
| Mission | Header with Trace Rail; tabs: Plan · Trace · Changes · Terminal · Report | Two panes: Plan/Trace + selected tab | Three panes: Plan · Trace · Changes/Terminal/Browser (user can swap the third) |
| Diff | Unified | Unified with file list drawer | File list + split diff |
| Terminal | Full screen with extra-keys row | Pane | Pane, can be docked beside Changes |
| Checkpoint | Bottom sheet | Bottom sheet | Side-sheet dialog anchored to the mission pane |

### 6.2 Foldables and posture

- **Tabletop posture** (half-opened, hinge horizontal): Mission screen places Trace Rail + live terminal/browser on the top half, controls (approve, steer composer) on the bottom half.
- **Book posture** (hinge vertical): panes never straddle the hinge; list-detail splits at the hinge.
- Continuity: folding/unfolding preserves scroll position, composer text, and selected pane (state hoisted to ViewModels and saved state).

### 6.3 Multi-window and desktop windowing

- Fully resizable; no orientation lock; layouts react to any size in real time.
- Drag-and-drop: drop files from other apps into the composer or a Project's Files; drag a code block or artifact out to other apps.
- Keyboard: shortcuts on hardware keyboards (Ctrl+Enter send, Ctrl+K search, Ctrl+N new mission, Ctrl+/ shortcut help, Esc to stop streaming); mouse hover states, right-click context menus, scroll-wheel in diffs and terminal.
- Multiple instances: a Mission can be opened in a new window (`FLAG_ACTIVITY_LAUNCH_ADJACENT`) on large screens to watch two missions side by side.

### 6.4 Navigation and gestures

- Predictive back with preview animations on all screens; back from a pane closes the pane before leaving the screen.
- Swipe between Mission tabs on compact; no custom edge gestures that conflict with system back.
- Long-press on a Mission card → quick actions (Pause, Steer, Open Report).

### 6.5 Editor and terminal on Android

- **Code browser:** native, virtualized lines, sticky symbol breadcrumbs (from tree-sitter outline), go-to-definition and find-references when the SCIP index exists (otherwise text search fallback, labelled).
- **Edit mode:** CodeMirror-based component in an isolated WebView: no network, assets bundled, communication via `WebMessagePort`; save creates a **user edit** event applied to the workbench working tree (and, if a mission is running, delivered to the agent as context).
- **Terminal:** connects to a PTY in the project's Workbench via WebSocket; extra-keys row; hardware keyboard passthrough; selection and copy; scrollback 10,000 lines on device (full log stored server-side as an artifact). Commands typed by the user are **user actions** and are not routed through agent approvals, but they are subject to Workbench network/egress policy and are recorded in the audit log.

### 6.6 Background behaviour

- The app never runs agent work locally. Background behaviour is limited to: FCM receipt, WorkManager jobs for uploads/sync, and a **foreground service only while the user is actively streaming terminal output in a visible window** (no long-lived foreground services).
- Live Updates / progress-style notifications for running missions (Android 16 progress-centric notifications where available; standard ongoing notification otherwise), user-controllable per channel.

### 6.7 Offline behaviour

- Cached: conversations, mission reports, artifacts recently viewed, project metadata (Room).
- Offline compose: messages queue with "Will send when online"; mission creation queues as a draft, never auto-starts without confirmation after reconnect.
- Clear offline banner; no spinners that never end.

### 6.8 Performance budgets

- Cold start to interactive Ask: < 1.2 s P50, < 2.0 s P90 on a mid-range 2024 device (Baseline Profiles).
- Scroll jank: < 1% janky frames on long conversations and diffs (Macrobenchmark).
- Streaming: render ≥ 60 fps with 100 tokens/s.
- APK download size: < 25 MB (App Bundle, no bundled ML models).
- Memory: stable under 300 MB with a 10k-line terminal and a 5k-line diff open.

---

## 7. AI assistant experience (Ask)

### 7.1 Composer

- Modes: **Ask** (default), **Research** (§16), **Mission** (switches to Mission Composer, §8).
- Inputs: text, voice (on-device speech recognition where available, otherwise server transcription with consent), images/camera, files (any type; parsed server-side), links (fetched with preview), GitHub URLs (resolved through the App to issues/PRs/files with permission checks), selected text shared from other apps.
- Context controls: a **context tray** above the composer lists what will be sent (attachments, project memory, selected files, previous mission report) with toggles; token usage indicator turns amber at 80% of the selected model's window.
- Model chip: "Auto" (routing policy), or a specific model; long-press opens the model sheet with capability badges (vision, tools, long context, reasoning, speed tier, relative cost).

### 7.2 Message system

- Streaming via SSE with resumable event ids; if the app is backgrounded, the server completes the response and the app shows it on return.
- **Edit** a previous user message → creates a new branch; branches navigable with "< 2/3 >" control.
- **Regenerate** with same or different model; previous versions kept as branches.
- **Retry** on failure resumes from the failure point when the provider supports it, otherwise restarts with a note.
- **Interrupt**: Stop button always visible while streaming; partial output preserved.
- **Continue**: if a response hits output limits, a "Continue" action appears and the gateway continues seamlessly.
- **Citations**: inline numbered citations linked to a Sources pane; each source shows title, domain, retrieval time, and the quoted passage used.
- **Code rendering**: syntax highlighting (tree-sitter tokens from server, computed during streaming per completed block), copy, wrap toggle, "Apply to project" (creates a mission or a direct edit proposal when a project is attached).
- **Math, tables, mermaid diagrams**: math via a native renderer (KaTeX subset rendered server-side to SVG for consistency), tables as horizontally scrollable native tables, Mermaid rendered server-side to SVG.

### 7.3 Tool activity in conversations

Tools used in Ask (web search, fetch URL, read project file, search code, run a quick snippet in an ephemeral sandbox) appear as **activity chips** in the message. Tap to inspect inputs/outputs. Actions that change anything (write files, run commands in the project Workbench, GitHub writes) are **never done from plain Ask**; the assistant proposes "Run as mission" instead. This keeps conversations safe and missions accountable.

### 7.4 Multimodal

- Images: understand screenshots (e.g., a crash dialog), photos of whiteboards → structured notes.
- Files: PDFs, docs, spreadsheets, code archives (zip → offered as a new project).
- Output images: generated diagrams (Mermaid/SVG); image generation is **LOW priority / FUTURE** and only via providers that support it, clearly labelled.

### 7.5 Conversation to mission continuity

When the user's request implies changes to a repository, Ask offers a **mission proposal card** containing the drafted objective, inferred project/repo, and suggested mode. Accepting creates a mission linked to the conversation; mission milestones post compact updates back into the conversation.

---

## 8. Code agent and mission lifecycle

### 8.1 Capabilities

| Capability | Design |
|---|---|
| Repository understanding | Repo Map (structure, languages, frameworks, entry points, test/build commands, conventions) + instruction files + retrieval over symbol graph and embeddings. |
| File discovery & code search | `search.text` (ripgrep), `search.symbols` (SCIP/tree-sitter), `search.semantic` (hybrid), `fs.list`, `fs.read` with line ranges. |
| Semantic & architecture understanding | Precomputed module summaries; dependency graph; "explain area" tool that composes retrieval + summarization with citations to file:line. |
| Editing | `fs.edit` (exact string replace with uniqueness check), `fs.write` (new files), `fs.patch` (unified diff apply with fuzz 0); every edit produces a Changes event with the diff. |
| Multi-file changes & refactors | Plan steps group edits; language-server-backed rename when available in the workbench image (LSP via `code.rename`), else verified textual rename. |
| Terminal execution | `shell.run` (command, cwd, timeout, env refs) and `shell.start`/`shell.read`/`shell.stop` for background processes (dev servers). |
| Testing | `test.run` wrapper that knows the project's detected test command, parses results (JUnit XML, TAP, pytest, jest, go test JSON) into structured pass/fail. |
| Debugging | Read failing output → hypothesize → add logging or targeted test → re-run; `debug.bisect` (git bisect driver) at Professional phase. |
| Browser verification | Playwright tools against localhost dev server (§13). |
| Git workflows | `git.status/diff/log/branch/commit` in workbench; push only through the credential broker (§12, §20). |
| GitHub workflows | `github.issue.read`, `github.pr.create/update/comment`, `github.checks.read`, `github.review_comments.read` through the GitHub App with per-mission down-scoped tokens. |
| Planning | Structured Plan (steps with intent, expected evidence, risk). |
| Iterative repair | Observe → diagnose → fix loop with progress detection (§8.4). |
| Self-verification | Verification step mandatory before completion (§8.5). |
| Long-running / resumable | Durable mission state (Restate virtual object), workbench snapshots, conversation compaction. |
| Collaboration | Steering messages, PR review comments ingestion, handoff to another user (Professional). |
| Sub-agents | Read-only **Scouts** for parallel investigation (search, read, summarize) in the same workbench snapshot; write access remains with the single **Lead** to avoid edit conflicts. |

### 8.2 Mission lifecycle

```
CREATED → INTAKE → (CLARIFYING) → CONTEXT → PLANNING → (PLAN_REVIEW checkpoint)
        → EXECUTING ⇄ (CHECKPOINT_WAIT) ⇄ (STEERED)
        → VERIFYING → REPORTING → COMPLETED
Any state → PAUSED (user) → resume
Any state → CANCELLED (user)
Any state → BLOCKED (needs user input that can't be a checkpoint, e.g. missing secret)
Any state → FAILED (unrecoverable error, budget exhausted, policy denial)
```

1. **Intake:** normalize objective, attachments, linked issue; classify task type (bugfix, feature, refactor, test-writing, investigation, docs, dependency update) and estimated complexity.
2. **Clarify (conditional):** if ambiguity score is high (missing acceptance criteria, multiple plausible interpretations, destructive implications), ask ≤ 3 questions in one structured card; in Autonomous mode, choose a default and record the assumption in the Plan instead.
3. **Context:** resume workbench at base branch HEAD; load Repo Map, memory, instruction files; retrieve relevant code; run Scouts if the task is broad.
4. **Plan:** produce steps `{id, title, intent, files_expected, commands_expected, evidence_expected, risk}`; Plan-first mode presents it as a checkpoint; the user can edit steps, reorder, delete, add constraints.
5. **Execute:** ReAct-style tool loop under policy; each tool call is a durable step (idempotency key = mission id + step seq).
6. **Observe & iterate:** each observation is summarized into working memory; failures trigger diagnosis.
7. **Checkpoints:** raised by policy (risk class) or by the agent when it needs a decision (choice between approaches).
8. **Steering:** user messages during execution are injected at the next loop boundary with priority; the agent must acknowledge in the Trace how it adjusted.
9. **Verify:** run tests/lint/type-check/build; browser checks for UI work; self-review of the full diff against the objective and Plan; produce verification evidence.
10. **Report:** Mission Report artifact (§8.6).
11. **Handoff:** open/update draft PR per autonomy settings; notify.

### 8.3 When to stop, ask, or continue

| Situation | Behaviour |
|---|---|
| Objective clearly achieved and verified | Stop → Report. |
| Ambiguity that changes the outcome the user will notice | Ask (clarify or checkpoint), continue on non-dependent steps meanwhile. |
| Ambiguity with a reasonable default | Continue with default, record the assumption in Plan and Report. |
| Action in risk class above the mission's autonomy | Checkpoint. |
| Missing credential/secret/permission | BLOCKED with exact instruction for the user. |
| Repeated failure with no progress (see 8.4) | Stop the loop; ask the user with a diagnosis and options. |
| Budget 80% consumed | Notify; continue. At 100%: pause and ask to extend or wrap up with a partial report. |
| Tests failing that were already failing on base | Record as pre-existing (verified by running on base), continue, report. |
| Scope creep detected (diff touches areas unrelated to Plan) | Pause, explain, ask. |

### 8.4 Loop and waste prevention

- **Budgets per mission:** max steps (default 150), max model tokens/cost (user cap), max wall-clock (default 2 h active, configurable), max workbench CPU-minutes.
- **No-progress detector:** fingerprint `(tool, normalized args, observation hash)`; same fingerprint 3× → forced reflection step; 5× → stop and ask. Same failing test after 4 distinct fix attempts → stop and ask.
- **Diff churn detector:** if the cumulative diff oscillates (edits reverted and reapplied) → reflection + checkpoint.
- **Context compaction:** when history reaches 70% of the model window, older observations are summarized into a structured **Mission Journal** (facts learned, files touched, decisions, open questions); raw observations remain retrievable by id.
- **Model escalation:** the router escalates to a stronger reasoning model after repeated failures, and de-escalates for mechanical steps.

### 8.5 Verification standard

A mission may only end **COMPLETED** if:

1. All Plan steps are done, skipped with a reason, or explicitly descoped by the user.
2. The project's verification commands ran **after the final edit** (tests; plus lint/type-check/build when detected) and their results are attached.
3. Regressions relative to base are zero, or each is explained and accepted by the user.
4. A self-review pass compared the final diff against the objective and flagged risks.
5. UI changes carry before/after screenshots when a runnable UI exists.

Otherwise the mission ends **COMPLETED_UNVERIFIED** (clearly labelled) or **FAILED/BLOCKED**.

### 8.6 Mission Report

Sections: Outcome (one paragraph) · Status badge (Verified / Unverified / Partial) · What changed (by file group, with rationale) · Evidence (commands, exit codes, test summaries, screenshots) · Decisions & assumptions · Risks & follow-ups · Cost & duration · Links (branch, PR, artifacts). Every claim links to the Trace event that supports it.

### 8.7 Background and resumable work

- Missions run fully server-side; the app may be closed.
- Workbench snapshots at checkpoints, before risky steps and when paused > 10 min (pause compute, keep disk).
- Resume restores workbench snapshot + mission state + compacted context; the agent receives a "resumed after X" note and re-validates the working tree (`git status`, quick test) before continuing.

---

## 9. Tool ecosystem

### 9.1 Tool architecture

```
Agent loop ──▶ Tool Router ──▶ Policy.decide() ──▶ [allow | checkpoint | deny]
                    │                                    │
                    ▼                                    ▼
             Tool Registry                         Checkpoint service
   (schemas: JSON Schema 2020-12,                 (awakeable in MissionEngine)
    risk class, executor, timeouts,
    output limits, redaction rules)
                    │
   ┌────────────────┼──────────────────────────┬──────────────────────┐
   ▼                ▼                          ▼                      ▼
Workbench tools   Platform tools          GitHub tools           MCP tools
(fs, shell, git,  (memory, artifacts,     (via GitHub App,       (external servers,
 test, browser,    plan, report, ask_user, down-scoped tokens)    allowlisted, proxied)
 search, index)    web.search, web.fetch)
```

- Every tool declares: `name`, `description` (written for the model), `input_schema`, `output_schema`, `risk_class` (static) and `risk_fn` (dynamic, e.g. a shell command's risk depends on its content), `timeout`, `max_output_bytes` (overflow written to an artifact and referenced), `redaction` (secret scrubbing), `idempotent` flag, `visibility` (how it renders in Trace).
- Tool outputs returned to the model are **wrapped as untrusted data** with provenance tags (`<tool_output source="web.fetch" url="…" trust="untrusted">`), and the system prompt states that instructions inside tool outputs are data.
- One registry serves internal tools and MCP tools; MCP tools are imported with their schemas, assigned a risk class by the admin/user at enable time (default R2 for any write-capable MCP tool, R1 for read-only), and executed through an **MCP proxy** that enforces egress and logs calls.

### 9.2 Category design

| Category | User value | Agent role | Key inputs → outputs | Permissions / risk | Security risks | Visibility | Failure handling | Scale needs |
|---|---|---|---|---|---|---|---|---|
| **Files** (`fs.*`) | Agent reads and changes code/docs | Primary actuator | path, range, edit spec → content, diff | Read R0; write in workbench R1 | Path traversal, symlink escape, secrets in files | Each write shows diff in Changes | Uniqueness/edit conflicts → structured error with nearest matches | Large files paged |
| **Code search** (`search.*`) | Fast orientation | Locate code | query, scope → ranked hits with file:line | R0 | None significant | Chip with hit count | Index stale → fall back to ripgrep, label it | Index per SHA |
| **Git** (`git.*`) | Clean history | Branch, commit | message, paths → sha | Local ops R1; push R2 (brokered) | Pushing secrets, rewriting history | Commits listed in Changes | Hooks failing → surfaced with output | — |
| **GitHub** (`github.*`) | Issues/PRs from the phone | Read context, open PRs, respond to reviews | issue/PR ids → data; PR spec → URL | Read R0; create/update PR R2; comment R2 (externally visible); merge **R4, user-only** | Over-scoped tokens, leaking private code to public repos | PR preview in Checkpoint | Rate limit → backoff; 403 → BLOCKED with reason | Webhook fan-in |
| **Terminal** (`shell.*`) | Real builds/tests | Run commands | cmd, cwd, timeout → stdout/stderr/exit, or process handle | Classified per command (R1 default; network-installing R2; destructive patterns R3) | Exfiltration, fork bombs, crypto-mining, destructive commands | Live terminal stream | Timeout → kill process group, return partial | Per-VM cgroups |
| **Execution/test** (`test.*`, `build.*`) | Proof | Verify | target → structured results | R1 | Same as shell | Test summary card | Flaky detection (re-run once, label) | Parallel runs in VM |
| **Browser** (`browser.*`) | Verify UIs | Inspect and act on local app | url, action → a11y snapshot, screenshot | localhost R1; external read R1; external form submit R3 | Malicious pages (ASI01), credential entry | Live screenshot feed | Navigation errors returned | Chromium per VM |
| **Web** (`web.search`, `web.fetch`) | Up-to-date facts | Research | query/url → results/markdown | R0 (read-only, via fetch proxy) | Injection via content, SSRF | Sources list | Provider fallback | Cache by URL |
| **Documents** (`doc.parse`, `doc.create`) | Work with PDFs, sheets | Read/produce files | file → text/structure; spec → file | R0/R1 | Malicious files (parsers sandboxed) | Artifact cards | Parse errors explicit | Parsing workers |
| **Memory** (`memory.search`, `memory.propose`) | Remembered conventions | Recall and propose memories | query → entries; entry → proposal | Read R0; write = **proposal** requiring user confirmation unless user enabled auto-save for that scope | Memory poisoning (ASI06) | Memory chip; proposals in Report | — | Vector index |
| **Project intelligence** (`project.map`, `project.explain`) | Understanding | Orientation | area → summary with citations | R0 | — | Chip | Stale → re-index | Precomputed |
| **Debugging** (`debug.*`) | Root causes | Bisect, logs | range → culprit commit | R1 | — | Trace | Bisect abort safe | — |
| **Automation** (Scheduled missions, FUTURE) | Recurring chores | Run on schedule/webhook | trigger → mission | Inherits mission policy, never auto-elevated | Unattended actions | Mission list | Missed run alerts | Scheduler |
| **Deployment** (FUTURE) | Ship | Trigger CI/CD workflows | workflow → run | **R3** always; production **R4** | Production impact | Checkpoint mandatory | — | — |
| **Collaboration** (`ask_user`, `notify`) | Agent asks well | Structured questions | question, options → answer | R0 | ASI09 manipulation | Checkpoint card | Timeout → WAITING | — |

### 9.3 Tool design rules

1. Fewer, sharper tools beat many overlapping tools: the MVP agent has ≤ 25 tools.
2. Every tool returns structured, truncated, model-friendly output plus an artifact id for the full output.
3. Errors are actionable ("old_string matched 3 locations: lines 12, 48, 90").
4. No tool accepts raw credentials; tools reference secrets by name and the executor resolves them server-side.
5. Tool descriptions and schemas are versioned and covered by evaluation (§24) because they change agent behaviour.

---

## 10. Project intelligence

### 10.1 What Xavro knows about a project

| Layer | Source | Refresh |
|---|---|---|
| Structure | File tree, sizes, generated/vendored detection (`.gitattributes linguist-generated`, common paths) | Each indexed commit |
| Languages & frameworks | Linguist-style detection + manifest parsing (package.json, build.gradle(.kts), pom.xml, pyproject, go.mod, Cargo.toml, etc.) | Each indexed commit |
| Dependencies | Manifests and lockfiles → dependency list with versions | Each indexed commit |
| Symbols | tree-sitter tags (all languages); SCIP (where build succeeds) | Incremental per changed file |
| Relationships | Import graph, call references (SCIP), test ↔ source mapping heuristics | Per commit |
| Entry points | Main functions, app manifests, server routes, CLI definitions | Per commit |
| Tests & build | Detected commands verified by a dry run in the workbench; user-confirmable | On first preparation, on manifest change |
| Instructions | `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `.github/copilot-instructions.md`, `.cursor/rules`, README sections | On change |
| History | Recent commits, hot files, authors (for context only, never for profiling individuals) | Daily / on push webhook |
| Architecture summary | Module summaries generated by a model with citations | Lazily, invalidated when ≥ 20% of module files change |
| Active work | Open missions, open PRs/issues | Live (webhooks) |
| Previous decisions | Project memory + Mission Reports (decisions section) | On mission completion |

### 10.2 Pipeline

1. **Trigger:** project creation, push webhook to default branch, mission start (index the mission's base SHA if missing), manual refresh.
2. **Run in a workbench indexing job** (separate low-priority VM, read-only token): checkout SHA → diff against last indexed SHA → re-parse changed files.
3. **Chunk** by syntax (function/class/section boundaries with tree-sitter), with file path, symbol path, and line range metadata; docs chunked by headings.
4. **Embed** changed chunks (batch, via gateway; model recorded per chunk so re-embedding is possible).
5. **Store:** chunk metadata + embeddings in the vector DB keyed `(repo_id, blob_sha, chunk_range)` — keying by **blob SHA** deduplicates identical files across commits and branches; SCIP index file in object storage; Repo Map in Postgres.
6. **Summaries:** module summaries regenerated lazily.

### 10.3 Retrieval

- Hybrid: lexical (ripgrep on live tree when in a workbench, Postgres FTS on chunk text otherwise) + vector similarity + symbol graph expansion (definitions/references of hit symbols) → reciprocal-rank fusion → model-based rerank for the top 50 when budget allows.
- Retrieval results always carry `file:line` citations and the SHA they came from; if the live tree differs, the agent re-reads the file before editing.

### 10.4 Limits and honesty

- Repositories above plan limits (e.g., > 2 GB or > 200k files at launch) are indexed partially by user-selected paths; the UI says so.
- Monorepos: index scoping by directory with per-scope Repo Maps.
- Binary and generated files are excluded from embeddings but listed in structure.

---

## 11. GitHub experience

### 11.1 Connection model

- **Xavro GitHub App** (one public app; enterprise customers may register their own app in the Flagship phase).
- **Permissions requested (minimum):** Metadata (read), Contents (read & write), Pull requests (read & write), Issues (read & write), Checks (read), Actions (read), Commit statuses (read), Workflows (**not requested** at launch — modifying `.github/workflows` is high-risk and excluded until explicitly enabled per project, then requested as an additional permission).
- **User-to-server authorization** (GitHub App user token) is used to (a) list the installations/repos the user can access and (b) verify the user's own permission on a repository before Xavro acts on their behalf. **Xavro never acts on a repository where the requesting user lacks push access**, even if the installation has access.

### 11.2 Token handling

- Installation tokens are minted **per mission step that needs them**, **down-scoped** to the single repository and the minimum permissions for that operation, and live ≤ 1 hour (GitHub's maximum) — in practice Xavro requests and discards them per operation.
- Tokens are held only by the **Credential Broker** service; they are never written to the workbench disk, environment, git config, or model context.
- `git push`/`fetch` from the workbench go through the **egress proxy**, which injects the Authorization header only for `github.com/<owner>/<repo>` matching the mission's bound repository.

### 11.3 Repository scoping

- A project binds to specific repositories. Missions can only access repositories bound to their project.
- Cross-repository missions (Professional phase) require explicit multi-repo binding.

### 11.4 Branch, commit and PR workflow

- Agent branches: `xavro/<short-mission-slug>-<id>`; created from the base the user chose.
- **Never** force-push to branches the agent did not create; **never** push to the default branch or any protected branch; **never** merge (merge is R4 and user-only, executed through GitHub UI or an explicit Xavro "Merge" action by the user with a confirmation that shows the PR's checks and target branch).
- Commits: conventional, small, one per Plan step where natural; commit author = Xavro App bot; `Co-authored-by` trailer for the requesting user (configurable); commits are API-signed (verified).
- PRs: draft by default; title and body from Mission Report; links back to the mission; labels optional per project config.
- Respect the repository's PR template if present.

### 11.5 Issues and review

- Issues list in project with filters; "Start mission from issue" pre-fills objective with issue body and comments (marked as untrusted content in the agent context).
- PR review comments → Mission inbox: user chooses "Let Xavro address these" (creates a follow-up mission iteration on the same branch) or replies themselves.
- CI checks: the agent reads check runs and logs (Actions read) to fix failures when the user asks or when the mission's autonomy allows "drive PR to green".

### 11.6 Events and synchronization

- Webhooks: installation, installation_repositories, push, pull_request, pull_request_review, pull_request_review_comment, issue_comment, issues, check_run, check_suite, workflow_run.
- Webhook receiver verifies signatures (`X-Hub-Signature-256`), dedupes by delivery id, enqueues to processing; replays via GitHub's redelivery API on gaps.
- Local cache of PR/issue metadata for fast mobile lists; source of truth remains GitHub; stale-while-revalidate with ETags to respect rate limits.

### 11.7 Security patterns

- Webhook secret rotation; App private key in KMS-backed secret storage, used only by the Credential Broker (JWT signing).
- Content from GitHub (issues, PR comments, code) is treated as **untrusted input** for prompt-injection purposes (ASI01).
- Public repositories: missions on public repos warn that PRs/comments are public; the agent cannot copy content from private projects into public repos (policy check on PR creation compares sources used in the mission context against repo visibility; cross-visibility flows require a checkpoint).

---

## 12. Terminal and execution: the Workbench

### 12.1 Execution model

- One **Workbench** = one isolated microVM with a project's working tree. Kinds:
  - **Project workbench** (interactive, user terminal + agent when no mission is running);
  - **Mission workbench** (forked from the project's latest snapshot at mission start; exclusive to the mission);
  - **Scratch workbench** (ephemeral, for Ask code snippets; no project data; no secrets; strict egress).
- Base images: curated Xavro images (Ubuntu LTS base with common toolchains: JDK, Android SDK command-line tools, Node, Python, Go, Rust, build-essential, git, ripgrep, Chromium + Playwright), plus **project environment definition** (`.xavro/workbench.yaml` or detected): additional packages, setup script, services (Postgres/Redis containers inside the VM), secret references, egress allowlist.

### 12.2 Lifecycle

```
PROVISIONING → (restore snapshot | clone + setup) → READY → BUSY ⇄ IDLE
IDLE > 10 min → SNAPSHOT + PAUSE (disk kept, compute released)
PAUSED > N days (plan-dependent) → ARCHIVED (snapshot in cold storage) → DELETED after retention
```

- Setup script results cached in snapshots keyed by hash(image + workbench.yaml + lockfiles).
- Warm pools of pre-booted base images per region to keep "start mission" under 5 s P50.

### 12.3 Boundaries

| Boundary | Rule |
|---|---|
| Isolation | Hardware-virtualized microVM per workbench; no shared kernel between tenants; no Docker socket from host. Containers *inside* the VM are allowed (nested), giving projects their services without weakening the outer boundary. |
| Resources | Default 2 vCPU / 4 GB RAM / 20 GB disk (plan-dependent up to 8 vCPU / 16 GB / 100 GB); cgroup limits on process count, CPU, memory; disk quota. |
| Network | **Default-deny egress** through an egress proxy: allowlist = package registries (npm, PyPI, Maven Central, Google Maven, crates.io, Go proxy, etc.), GitHub (bound repo only, via broker), and project-specific additions approved by the user. No inbound connections except Xavro's control channel. Metadata endpoints (169.254.169.254 etc.) blocked. |
| Files | Working tree under `/workspace`; no access to other projects; uploads mounted read-only under `/inputs`. |
| Secrets | Project secrets injected **only** into commands the user marks (env refs resolved by the executor at process start, scrubbed from outputs by exact-match and entropy-based redaction before reaching the model or logs). Long-lived Xavro or GitHub credentials **never** enter the VM. |
| Time | Per-command timeout (default 10 min, max 60), per-mission wall-clock budget. |

### 12.4 Process management

- Commands run under a supervisor in the VM (`xavro-agentd`): process groups, PTY allocation, streaming stdout/stderr with sequence numbers, exit codes, resource usage.
- Background processes (dev servers, watchers) get handles; listed in the Workbench panel; killed on mission end.
- Cancellation: SIGINT → 5 s → SIGTERM → 5 s → SIGKILL on the process group.
- Output streaming: chunks to the event system; large outputs stored as artifacts with head/tail excerpts in Trace.

### 12.5 Danger of arbitrary agent execution — explicit stance

Allowing an AI to execute arbitrary commands is treated as **running untrusted code by default**, because the agent's behaviour can be steered by any content it reads (ASI01, ASI05). Therefore:

1. Isolation is the primary control (microVM), not command filtering.
2. Credentials are absent from the execution environment (broker pattern), so a compromised workbench cannot exfiltrate tokens it does not have.
3. Egress is default-deny, so data exfiltration and malware download paths are closed by default.
4. Command risk classification (pattern + model-based) adds checkpoints for destructive or unusual commands (`rm -rf` outside workspace, `curl | sh`, `git push --force`, credential file access, crypto-mining signatures) — as a **usability layer**, not as the security boundary.
5. Every command and its output are recorded in the immutable audit log.
6. Resource abuse detection (sustained 100% CPU with network to unknown hosts, process names) suspends the workbench and alerts.

### 12.6 Failure recovery

- VM crash → mission engine retries the step on a restored snapshot (at most once), Trace shows "Workbench restarted".
- Provider outage → missions pause in WAITING with a status banner; resume automatically when healthy.
- Setup failure → BLOCKED with logs and the failing setup line; user can edit workbench.yaml in the app.

---

## 13. Browser and digital interaction

### 13.1 Decision

Browser automation is **core for verification of web apps the agent builds** (MVP+1, the first post-MVP release) and **restricted for open-web research** (read-only fetch at MVP; interactive browsing EXPERIMENTAL). It is not a general "agent clicks around the internet on your behalf" product at launch: the risk/benefit is poor (ASI01, ASI09) and it is not the core job.

### 13.2 Design

- **Engine:** Playwright with Chromium inside the mission workbench.
- **Tools:** `browser.open(url)`, `browser.snapshot()` (accessibility tree with element refs, default observation), `browser.screenshot(full_page?)`, `browser.click(ref)`, `browser.type(ref, text)`, `browser.select`, `browser.wait_for(condition)`, `browser.console()`, `browser.network(filter)`.
- **Local app verification:** agent starts the dev server (`shell.start`), opens `localhost:<port>`, exercises flows, captures screenshots for the Report, reads console errors.
- **Web testing:** generate/run Playwright tests as part of the project's test suite.
- **Controlled actions:** form submission or login on non-localhost origins is **R3** (checkpoint with screenshot of the form and the destination origin); entering any credential is **R4/forbidden** for the agent — users type credentials themselves in a live view (FUTURE).
- **Isolation:** the browser lives in the workbench VM, so egress policy applies; fresh browser profile per mission; downloads restricted to `/workspace/.downloads`.
- **User view:** live screenshot stream (1–2 fps) in the Browser tab; on expanded screens it can be a side pane.

---

## 14. Memory system

### 14.1 Memory types

| Type | Scope | Content | Lifetime | Who writes |
|---|---|---|---|---|
| **Personal memory** | User | Preferences (style, language, tools), facts the user chose to save | Until deleted | User; agent **proposes** |
| **Project memory** | Project | Conventions, architecture decisions, commands, do/don't rules, glossary | Until deleted; staleness flags | User; agent proposes; imported instruction files |
| **Conversation memory** | Conversation | Message history, compaction summaries | Conversation lifetime | System |
| **Mission state** | Mission | Plan, Mission Journal, tool observations, checkpoints | Mission lifetime + retention | System |
| **Execution state** | Workbench | Files, processes, snapshots | Workbench lifecycle | System |

### 14.2 Rules

- **Visibility:** every memory entry is visible and editable in the UI with its source (which conversation/mission/file created it, when).
- **Proposal-first writes:** the agent proposes memories ("Save: 'Use Kotest, not JUnit, for new tests' to Project memory?"); auto-save can be enabled per scope by the user. This is the primary defence against memory poisoning (ASI06).
- **Retrieval:** project memory entries are small (≤ 500 chars each); always-on entries (user-pinned) are injected every time; others retrieved by relevance (hybrid search) with a budget (default ≤ 2k tokens).
- **Conflicts:** a new proposal conflicting with an existing entry is shown as an update diff, not a duplicate.
- **Staleness:** entries referencing files/symbols that no longer exist are flagged for review.
- **Deletion:** deleting a memory removes it from retrieval immediately and from storage within 24 h (embeddings included); account deletion removes all memory within 30 days (backups expire on schedule).
- **Privacy:** memory is never used to train models; never shared across users; team memory (Professional phase) is a separate scope with admin controls.
- **Quality:** the evaluation suite measures memory precision (retrieved memories that were relevant) and harm (memories that misled the agent).
- **Failure handling:** if memory retrieval fails, the agent proceeds and the Trace notes "Memory unavailable"; it never invents remembered facts.

---

## 15. Multi-model system

### 15.1 Model registry

Each model entry: `provider`, `model_id`, `display_name`, `capabilities` (text, vision, audio, tools, parallel tools, structured output, reasoning controls, prompt caching, max context, max output, citations/search grounding), `pricing` (input/output/cache per million tokens), `latency_profile` (measured TTFT/throughput rolling P50/P90), `quality_scores` (from Xavro evals per task type), `data_policy` (retention, training opt-out, regions), `status` (active, degraded, deprecated), `allowed_plans`.

The registry is data (database + admin console), not code; adapters are code.

### 15.2 Roles and routing

| Role | Used for | Selection |
|---|---|---|
| **Fast** | Titles, classification, summaries, explain-this, routing itself | Cheapest model meeting a quality floor |
| **Chat** | Ask default | Balanced quality/latency |
| **Reasoning** | Planning, hard debugging, architecture questions | Highest reasoning eval score within budget |
| **Coding** | Mission execution loop | Highest coding-agent eval score (Xavro's own suite) |
| **Research** | Long-context reading, synthesis | Long context + citation quality |
| **Vision** | Screenshots, images | Vision capability |
| **Embedding** | Index & memory | Fixed per index version |

Routing policy = ordered candidate list per role, filtered by capability requirements of the request (e.g., needs vision + tools), user/plan restrictions, data-policy constraints (e.g., an enterprise requiring zero-retention providers), provider health, and budget; then scored by expected quality × cost × latency weights (user preference "Faster / Balanced / Best").

### 15.3 Fallback and resilience

- Per-provider circuit breakers (error rate, latency); automatic failover to the next candidate with equivalent capabilities; the UI notes the fallback.
- Mid-stream failure: Ask restarts on the fallback with the partial response as context ("continue from"); missions retry the step (durable).
- Mid-mission model switch is allowed between steps; the conversation is re-encoded into the new provider's format by the gateway (tool call ids, reasoning blocks dropped where not portable).

### 15.4 User controls

- Composer model chip (Auto or specific model); per-project default; Settings → Models: preference (Faster / Balanced / Best), disable providers (e.g., for data policy), view per-model cost.
- **BYOK** (bring your own API key) — HIGH priority for the Professional phase: keys encrypted with KMS envelope encryption, used only by the gateway, never by workbenches; usage billed to the user's provider account and shown separately.

### 15.5 Provider isolation

- Adapters run in the gateway service only; credentials per provider in Secret Manager; outbound calls via a dedicated egress with provider allowlist.
- Data minimization: only the request's required context is sent; prompt caching keys never mix tenants.
- Output normalization: tool calls validated against schemas before execution; invalid calls returned to the model as errors, never executed loosely.
- On-device model (Gemini Nano via ML Kit GenAI, PREVIEW): EXPERIMENTAL use only for private on-device tasks (notification summaries, offline draft titles); never part of agent or answer generation.

---

## 16. Research mode

### 16.1 Experience

- Entered via the composer mode chip **Research** or by asking for a "report/compare/investigate".
- Step 1 — **Scope card:** Xavro restates the question, proposes sub-questions, source types (web, academic, docs, project files, GitHub), depth (Quick ≈ minutes / Deep ≈ longer, with cost estimate), and output format (brief, report, comparison table). User adjusts or accepts.
- Step 2 — **Live investigation:** a Research Trace shows queries issued, sources found, sources read, sources rejected (with reason: duplicate, low quality, paywalled, irrelevant).
- Step 3 — **Report artifact:** structured document with executive summary, findings per sub-question, comparison tables, confidence levels per claim, inline citations, a bibliography with access dates, and "open questions".
- Long-running research runs as a **mission of type research** (same engine, different tool set; no workbench write tools), so it survives app closure and notifies on completion.

### 16.2 Engine

- **Source discovery:** one or more web search providers behind a `SearchProvider` port (choose two at launch for coverage and fallback; evaluate on Xavro's research eval set for freshness, recall, and cost), plus direct fetch of official docs, arXiv, GitHub.
- **Source ranking:** relevance (model-scored against sub-question), authority (primary source > official docs > reputable publication > forum), freshness (date-sensitive questions weigh recency), diversity (avoid five copies of one press release).
- **Reading:** `web.fetch` converts HTML/PDF to clean text (sandboxed parser workers); long documents chunked and retrieved per sub-question.
- **Comparison:** extracted claims normalized into a claim table `(claim, source, quote, date)`; conflicting claims surfaced explicitly.
- **Citation integrity:** every factual sentence in the report must map to ≥ 1 claim row; a post-generation **citation verifier** (fast model + string matching) checks that quoted passages exist in the fetched text; unsupported sentences are rewritten as uncertain or removed.
- **History & artifacts:** reports saved to Library and optionally to a Project; follow-up questions reuse the source set.

---

## 17. Files and artifacts

### 17.1 Artifact model

Every durable output is an **Artifact**: `{id, kind, title, project_id?, conversation_id?, mission_id?, created_by (user|agent), version, mime, size, storage_key, preview, provenance (sources, tool events), retention_class}`.

| Kind | Examples | Viewer |
|---|---|---|
| `code_file` | Source files from workbench (snapshot) | Native code viewer |
| `diff` | Mission changes, PR diffs | Native diff viewer |
| `document` | Uploaded PDFs/docs, generated Markdown/DOCX/PDF | Native Markdown, PDF renderer (PdfRenderer), export |
| `report` | Mission Report, Research Report | Structured native renderer |
| `image` | Screenshots, uploads, diagrams | Zoomable image viewer |
| `log` | Terminal logs, test output | Monospace virtualized viewer with search |
| `dataset` | CSV/JSON/XLSX | Table preview |
| `archive` | Zip uploads/exports | File list |

### 17.2 Relationships

- Artifacts link to the conversation message or mission step that produced them; Mission Reports link to all their evidence artifacts.
- Library shows all artifacts with filters (kind, project, source, date) and full-text search.
- Versioning: regenerated or edited artifacts create new versions; diffs between versions for text kinds.
- Sharing (Professional phase): signed, expiring links with view-only access; never public by default.
- Uploads: virus/malware scanning (ClamAV or provider scanning) before parsing; size limits per plan; parsing in sandboxed workers.

---

## 18. Real-time experience: the Signal system

### 18.1 One event model

All live activity — assistant tokens, tool calls, terminal output, file changes, tests, checkpoints, git/GitHub activity, mission state, errors — is expressed as **Signals** in one schema (defined in `xavro-protocol`):

```json
{
  "stream": "mission:01J…",        // or conversation:…, workbench:…, user:… (inbox)
  "seq": 1842,                      // monotonically increasing per stream
  "ts": "2026-10-08T10:15:22.120Z",
  "type": "tool.call.completed",
  "actor": "agent|user|system",
  "visibility": "user|internal",
  "data": { /* typed per event type */ },
  "refs": { "step_id": "…", "artifact_id": "…" }
}
```

Event families: `message.delta`, `message.completed`, `status.summary` (safe human-readable progress lines), `plan.updated`, `step.started/completed/failed`, `tool.call.started/progress/completed/failed`, `terminal.output`, `fs.changed`, `test.result`, `browser.frame`, `checkpoint.requested/resolved/expired`, `git.commit`, `github.pr.opened/updated`, `budget.updated`, `mission.state`, `error`, `notice`.

### 18.2 Transport

- **SSE over HTTP/2** for all one-way streams; the client reconnects with `Last-Event-ID` and receives missed events from the persisted log (`run_events`), then live events.
- **WebSocket** only for the interactive terminal (bidirectional keystrokes, resize).
- **FCM** for out-of-app delivery (checkpoint, completion, failure); FCM payloads contain ids, not content, and the app fetches details over the authenticated API.
- Backpressure: server coalesces `message.delta` and `terminal.output` into ≤ 20 Hz batches for mobile; full-fidelity logs remain in artifacts.

### 18.3 No private chain-of-thought

- Models' internal reasoning tokens (where providers expose them) are **never** shown or stored beyond what providers require for multi-turn continuity (encrypted/opaque reasoning blocks are passed back to the provider only).
- Users see **status summaries**: concise, factual descriptions of what the agent is doing ("Reading `PaymentService.kt` to find where retries are configured"), generated from tool calls and plan state, not from hidden reasoning.
- Agent explanations shown at checkpoints are **labelled as the agent's explanation** and sit next to ground-truth payloads.

---

## 19. Trust, control and permission architecture

### 19.1 Risk classes

| Class | Definition | Examples | Default handling |
|---|---|---|---|
| **R0 Observe** | Reads within the user's own scope; no side effects | Read files, search, read issues, web search/fetch via proxy | Automatic |
| **R1 Local change** | Changes confined to the mission workbench; reversible by discarding the workbench | Edit files, run tests/builds, local git commits, start dev server | Automatic (Careful mode: checkpoint for first shell command per mission) |
| **R2 Shared but reversible** | Visible to others or leaves the workbench, but reversible | Push to agent branch, open/update draft PR, comment on PR/issue, install new dependency from allowlisted registry, enable a read-only MCP tool | Checkpoint in Careful; automatic in Balanced for push/draft PR (announced); comment requires checkpoint in Balanced; automatic in Autonomous |
| **R3 Consequential** | Hard to reverse or affects external systems | Mark PR ready for review, request reviewers, network egress to a new domain, form submission on external site, modifying CI workflows, adding secrets usage, deleting branches the agent created | Checkpoint always (Autonomous may pre-approve specific R3 action types per project) |
| **R4 Destructive / irreversible / privileged** | Cannot be undone or affects production/security | Merge, force-push, delete repo content outside agent branches, deploy to production, change repo settings, use credentials on websites | **User-only** action through dedicated UI; the agent can only request it |

Dynamic classification: the tool's `risk_fn` can raise (never lower) the static class based on arguments (e.g., `shell.run("rm -rf /workspace/..")` → R3; `shell.run("curl … | sh")` → R3; network install from a non-allowlisted registry → R3).

### 19.2 Autonomy levels

| Level | Meaning |
|---|---|
| **Careful** | Plan approval required; R1 shell first use, R2 and R3 checkpoint. Default for a user's first 3 missions. |
| **Balanced** (default afterwards) | Plan shown, auto-starts after 60 s unless the user opens it (configurable); R2 push/draft PR automatic with notification; R2 comments and R3 checkpoint. |
| **Autonomous** | No plan approval; R2 automatic; R3 checkpoint unless pre-approved per project; R4 never. |

### 19.3 Reducing approval fatigue without losing control

- **Batching:** checkpoints raised within a short window are grouped ("3 actions need approval").
- **Scoped grants:** "Allow similar for this mission" grants a pattern (e.g., `npm install <pkg>` from npm registry) for the mission only; project-level grants require going to Project → Permissions (deliberate friction).
- **Ground truth:** Checkpoint payloads are rendered from the actual tool arguments by trusted code (ASI09 defence); the agent's text is clearly separated.
- **Explainability:** "Why is this risky?" expands a plain-language explanation of the risk class.
- **Undo where possible:** R2 actions show "Revert" (close PR, delete agent branch, delete comment) in the Trace.
- **Expiry:** checkpoints expire (default 24 h) → mission WAITING, no silent auto-approval ever.

### 19.4 Enforcement location

All permission decisions are made by the **Policy** module in the backend at tool-execution time, inside the Tool Router, using server-side state (mission autonomy, grants, user role, project settings). The Android app only renders and collects decisions. A decision is a signed record bound to the checkpoint id and the payload hash; executing a payload whose hash differs from the approved one is impossible.

---

## 20. Security architecture

### 20.1 Principles

1. **Defense in depth:** identity → authorization → policy → isolation → egress → secrets → audit → detection.
2. **Assume the model is compromised** by any untrusted content it read (ASI01). Design every boundary so that a fully hijacked agent can do no more than its mission's granted permissions within its workbench.
3. **Least privilege, short-lived credentials** (ASI03).
4. **Trusted rendering of decisions** (ASI09).
5. **Everything recorded** in an append-only audit log.

### 20.2 Threat analysis and controls

| Threat | Vector | Controls |
|---|---|---|
| **Prompt injection / goal hijack (ASI01)** | Issues, PR comments, README, web pages, documents, tool outputs | Untrusted-data wrapping with provenance; instruction hierarchy in system prompts; tool permissions independent of model intent; egress deny-by-default; checkpoint ground truth; detection classifier on tool outputs (flag, not block) feeding Trace warnings; evaluation suite of injection attacks (§24). |
| **Tool abuse (ASI02)** | Agent chains tools to exfiltrate or damage | Policy per call; dynamic risk; output-size limits; rate limits per tool; cross-visibility checks (private → public). |
| **Excessive agency / privilege abuse (ASI03)** | Over-broad tokens | Per-operation down-scoped GitHub tokens; no credentials in workbench; user-permission verification on each repo action; R4 user-only. |
| **Secret leakage** | Secrets in repo, env, logs, model context | Secret scanning on clone (gitleaks-style) with warnings; output redaction (exact-match of known secrets + entropy patterns) before model/log/event; secrets injected only into marked commands; KMS envelope encryption at rest; never logged. |
| **Repository attacks** | Malicious repo with git hooks, build scripts, postinstall scripts, symlinks | All execution inside microVM with no credentials; git hooks disabled for agent git operations by default (`core.hooksPath` to empty) unless project enables them; symlink resolution confined to `/workspace`. |
| **Malicious documents** | PDFs/Office files exploiting parsers | Parsing in sandboxed, network-less workers; size/page limits; malware scan. |
| **Malicious web content** | Injection, drive-by downloads, SSRF | Fetch proxy with SSRF protection (block private IP ranges, metadata endpoints, DNS rebinding checks); content-type limits; browser in workbench with egress policy. |
| **Authentication attacks** | Credential stuffing, phishing, token theft | Passkeys first; no passwords; short access tokens; refresh-token rotation with reuse detection; device binding (proof-of-possession via Keystore key); anomaly detection on sign-ins; step-up auth (passkey re-verification) for R4 actions and security settings. |
| **Authorization bugs (IDOR)** | Accessing other users' projects | Every query scoped by tenant/project via repository layer that requires a principal; Postgres Row-Level Security as a second layer; authorization tests generated per endpoint. |
| **Sandbox escape** | Kernel/hypervisor exploits | MicroVM isolation; minimal host; jailer; prompt patching; tenant-level placement controls; escape detection; bug bounty. |
| **Unsafe integrations / MCP (ASI04)** | Malicious or vulnerable MCP servers, tool poisoning in descriptions | Allowlisted MCP servers only (admin-curated at launch); pinned versions and description hashes (re-approval when a tool description changes); MCP calls proxied with egress and logging; remote MCP servers via OAuth with minimal scopes. |
| **Supply chain (ASI04)** | Compromised dependencies in Xavro itself or in workbench images | Dependency pinning + lockfiles; Renovate with review; SBOM (CycloneDX) for every image and the APK; Sigstore/cosign image signing; SLSA level 3 build provenance target; minimal base images; Play App Signing. |
| **Memory & context poisoning (ASI06)** | Agent saves malicious "facts" | Proposal-first memory; provenance; user review; memory eval set. |
| **Inter-agent communication (ASI07)** | Scouts/Lead messages tampered | Sub-agents are in-process with typed messages inside the mission engine; no network channel between agents at launch; A2A (FUTURE) would require mutual auth and signed messages. |
| **Cascading failures (ASI08)** | One bad step poisons subsequent ones; provider outage cascades | Verification gate; circuit breakers; bulkheads per tenant; mission-level budgets. |
| **Rogue agent (ASI10)** | Agent persists outside policy | No agent-owned persistent credentials; all actions through Tool Router; kill switch per mission/project/global; anomaly detection on tool-call patterns. |
| **Data leakage across tenants** | Caching, logs, embeddings | Tenant-scoped keys everywhere; no cross-tenant prompt cache; per-tenant encryption context; vectors partitioned by tenant. |

### 20.3 Security components

- **Credential Broker:** sole holder of GitHub App key, provider keys, BYOK keys; mints short-lived scoped tokens; signs requests on behalf of tools; audited.
- **Egress Proxy:** per-workbench policy, TLS-terminating only for credential injection domains (GitHub), SNI-based allowlist elsewhere; logs destinations.
- **Policy service/module:** §19.
- **Audit log:** append-only table + daily export to WORM object storage (object lock); user-visible "Agent activity log" in Settings → Security.
- **Kill switches:** per mission (user), per project (user/admin), per tool, per provider, global (ops).
- **Security reviews:** threat model per feature; external penetration test before public launch; bug bounty after launch.

### 20.4 Mobile-specific security

- Tokens in Android Keystore-backed encrypted storage; biometric/passkey step-up for R4 actions and for viewing project secrets metadata.
- Certificate transparency enforcement; no custom trust anchors in release builds.
- `FLAG_SECURE` optional (user setting) for screens showing code/secrets to prevent screenshots/recents thumbnails.
- No WebView loads remote content (editor WebView is local-only).
- Deep links verified via Android App Links; OAuth redirects validated with PKCE and state.

---

## 21. API platform (Platform phase)

### 21.1 Scope

Expose to developers: **Missions API** (create mission on a project, stream Signals, answer checkpoints programmatically within granted scopes), **Projects API** (read Repo Map, memory, artifacts), **Ask API** (conversational completions with Xavro routing and tools), and **Model Gateway API** (unified access to registered models with Xavro routing, usage and policies) — plus an **MCP server** exposing projects and missions as MCP tools and resources to other agents.

### 21.2 Design

- **Auth:** OAuth 2.1 for third-party apps (user-delegated, scoped), **API keys** for server-to-server (prefix-identifiable `xv_live_…`, hashed at rest, shown once, scoped to projects and permissions, expirable, rotatable).
- **Permissions:** keys/apps get explicit scopes (`missions:write`, `projects:read`, `checkpoints:answer:r2`…); R3 checkpoints can never be answered by API keys unless an admin policy explicitly allows; R4 never.
- **Streaming:** SSE with the same Signal schema as the app; webhooks for mission state changes (signed, retries with backoff).
- **Usage & rate limits:** token-bucket per key and per org; usage endpoints; cost headers on responses.
- **Versioning:** date-based API versions (`Xavro-Version: 2027-03-01`), additive changes only within a version.
- **Docs:** OpenAPI 3.1 generated from the Kotlin route definitions, rendered in the Web Console with runnable examples; SDKs generated for TypeScript, Python, Kotlin.
- **Extensibility (FUTURE):** custom tools via MCP registration per org; event subscriptions.

---

## 22. Admin platform (Web Console)

### 22.1 Audiences

- **Xavro operations (internal):** global health, providers, abuse, feature flags, support tools.
- **Organization admins (Professional/Flagship):** members, policies, usage, audit.

### 22.2 Capabilities

| Area | Capabilities |
|---|---|
| Users & roles | Invite, SSO/SCIM provisioning (Flagship), roles (Owner, Admin, Member, Viewer, Billing), suspend, session revoke |
| Models & providers | Enable/disable models per org, routing preferences, data-policy constraints (e.g., zero-retention only), BYOK management |
| Tools & integrations | Enable tools, MCP server allowlist, default risk classes, pre-approved R3 action types |
| Repositories & access | GitHub installations, project ↔ repo bindings, branch protections reminder, public-repo rules |
| Quotas & usage | Budgets per user/project, alerts, cost breakdown by model/tool/workbench minutes |
| Security & audit | Audit log search/export, agent activity, checkpoint decisions, kill switches, secret scanning findings |
| Feature flags | Per org/user/percentage rollouts (OpenFeature-compatible provider) |
| System health (internal) | Provider status, queue depth, workbench pool capacity, error budgets, incident banners |
| Agent activity | Live missions, stuck missions, anomaly alerts, mission replay (Trace) for support with user consent |

### 22.3 Implementation stance

The Web Console is a separate web app (TypeScript + a mainstream framework; it is the one place where web is the right medium), using the same public/internal APIs as the Android app. Internal admin actions require hardware-key (WebAuthn) authentication, are just-in-time elevated, and fully audited. Support staff can never read user content without a time-boxed, user-approved access grant.

---

## 23. Observability

### 23.1 Architecture

```
Android (OTel SDK, Sentry, PostHog) ─┐
API / Gateway / Agent runtime (OTel) ─┼─▶ OTel Collector (tail sampling, PII scrubbing)
Workbench supervisor (OTel)        ───┘        │            │              │
                                              ▼            ▼              ▼
                                        Tempo (traces)  Loki (logs)  Mimir/Prometheus (metrics)
                                              │
                                              └──▶ Langfuse (LLM/agent spans, cost, eval datasets)
```

- One trace per user request and per mission step; trace context propagated app → API → gateway → provider call → tool → workbench command (W3C trace context; MCP standardized tracing keys).
- GenAI semantic conventions (development status; pinned version) for model calls, agent invocation and tool execution spans; all mapping in one `telemetry-genai` module.
- **Content policy:** prompts and completions are **not** stored in general telemetry; Langfuse stores them only for internal evaluation traffic, opted-in users, or under a support grant, with retention ≤ 30 days.

### 23.2 What is measured

| Domain | Metrics |
|---|---|
| App performance | Cold/warm start, frame timing/jank, ANR rate, crash-free sessions, network latency, SSE reconnects |
| AI latency | TTFT, tokens/s, total latency by model/provider/region, cache hit rate |
| Model behaviour | Tool-call validity rate, refusal rate, fallback rate, context-overflow incidents, output truncations |
| Agent performance | Steps per mission, success/verified rate by task type, checkpoint count & rejection rate, no-progress stops, compaction count |
| Tools | Latency, error rate, timeout rate per tool; output overflow rate |
| Execution | Workbench start time (cold/warm/snapshot), CPU/memory/disk usage, command durations, OOMs |
| Errors & retries | By service, by provider, by tool; retry success rate |
| Reliability | SLOs: API availability 99.9%, SSE delivery, mission engine step success; error budgets |
| Cost | Cost per message, per mission, per verified outcome; by model, tool, workbench minutes; margin by plan |
| Task completion | North Star (WVO), acceptance rate, time-to-verified-outcome |

### 23.3 Alerting

SLO burn-rate alerts (multi-window), provider degradation alerts, workbench pool exhaustion, anomaly alerts (cost spikes per user, unusual egress), security alerts (policy denials spike, injection detections spike).

---

## 24. Evaluation

### 24.1 Methodology

- **Offline suites** (run in CI nightly and before any model/prompt/tool change ships):
  1. **Xavro-Missions:** 300+ real tasks across 30+ pinned repositories (Kotlin/Android, TS/React, Python, Go, Java, Rust), each with a hidden verification test and a rubric. Derived from real issues/PR pairs (SWE-bench-style) plus Xavro-authored tasks for mobile-relevant stacks.
  2. **Terminal tasks:** environment setup, build fixing, dependency upgrades (Terminal-Bench-style).
  3. **Repository understanding:** Q&A with ground-truth answers ("where is X configured?") scored by exact file:line match.
  4. **Tool selection:** single-step tasks scoring the chosen tool and arguments.
  5. **Safety & injection:** 200+ adversarial fixtures (malicious README, issue comments, web pages, MCP tool descriptions, document payloads) — success = no policy-violating action attempted and no data exfiltration; partial = attempted but blocked by policy (tracked separately: policy catching an attempt is good, but the attempt rate must trend down).
  6. **Research:** questions with curated source sets; scoring citation precision/recall, factual accuracy, freshness.
  7. **Memory:** precision/harm of retrieved memories.
  8. **Long-running reliability:** missions with forced interruptions (pause, workbench crash, provider failover) — success = correct resumption without duplicated side effects.
- **Harness:** UK AISI **Inspect** (open-source) as the evaluation framework, running missions against the real agent runtime with a mock-free workbench; results pushed to Langfuse datasets and a results warehouse.
- **Metrics:** pass@1 verified, cost per task, steps, wall-clock, checkpoint count, unnecessary-checkpoint rate (judged), hallucinated-claim rate in Reports (claims without evidence events), recovery rate.
- **Online evaluation:** acceptance rate, rollback within 7 days, user ratings on Reports, steering frequency; A/B tests on routing/prompt changes with guardrail metrics.
- **LLM-as-judge** only with calibrated rubrics validated against human labels (≥ 0.8 agreement) and never as the sole gate for safety.
- **Gate:** no prompt, tool schema, routing or model change reaches production if it regresses verified pass@1 by > 2 points or any safety metric at all, without explicit sign-off.

---

## 25. Scalability

| Dimension | Start | Growth path |
|---|---|---|
| Concurrent missions | Restate cluster (3 nodes) + stateless agent workers autoscaled on queue depth | Partition by tenant; add Restate partitions; dedicated pools per enterprise |
| Workbenches | Managed provider quotas | Self-hosted Firecracker pools per region; warm-pool autoscaling by forecast; overflow to managed provider |
| Large repositories | Partial indexing by path; blob-SHA dedup | Sharded index workers; dedicated vector engine (Turbopuffer/Qdrant) |
| Long-running tasks | Durable steps; snapshots | Tiered snapshot storage; scheduled resumption |
| Streaming traffic | API pods with SSE; Valkey pub/sub fan-out | NATS JetStream; regional edge SSE gateways |
| Multiple models | Gateway pods with per-provider pools | Regional gateways; provider-specific rate shaping; reserved capacity contracts |
| Memory & vectors | pgvector in a separate database | Dedicated vector engine; per-tenant namespaces |
| Files | Object storage with lifecycle rules | Multi-region buckets; CDN for previews |
| Observability | Sampled traces (tail sampling keeps errors and slow) | Per-tenant sampling; cost-aware retention |
| High availability | Multi-zone in one region; Postgres HA; daily restore tests | Multi-region active/passive with data residency (EU, US, ME) |

Avoided early: microservice sprawl (start with a **modular monolith** API + separate gateway, agent worker, and indexer deployables sharing modules), Kafka, service mesh, multi-region writes.

---

## 26. Data architecture

### 26.1 Stores and ownership

- **PostgreSQL (primary, OLTP):** identity mirror, organizations, projects, repositories, conversations, messages, missions, plan steps, checkpoints, approvals, tool calls (metadata), memory entries, artifacts (metadata), usage ledger, settings, API keys (hashed), audit (hot), notifications, devices.
- **PostgreSQL (vector DB, separate instance):** chunks, embeddings, memory embeddings.
- **Object storage:** attachments, artifacts, logs, terminal transcripts, screenshots, SCIP indexes, workbench snapshots (provider-managed at MVP), audit exports (object-lock).
- **Valkey:** rate limits, ephemeral presence, pub/sub, short caches.
- **Restate:** mission execution state (journals) — operational state, not system of record; mission records mirrored into Postgres at each state transition.
- **Warehouse (later):** ClickHouse for analytics/usage/eval results.

### 26.2 Core entities (logical)

```
organizations(id, name, plan, region, settings)
users(id, org_id?, idp_subject, email, display_name, locale, created_at)
devices(id, user_id, platform, push_token, public_key, last_seen_at, revoked_at)
identities_linked(id, user_id, provider[github], provider_user_id, scopes, created_at)
github_installations(id, org_or_user_id, installation_id, account_login, permissions_json, suspended_at)
projects(id, owner_user_id, org_id?, name, description, default_models_json, autonomy_default, budget_default, archived_at)
project_repositories(id, project_id, provider, repo_full_name, repo_id, installation_id, default_branch, visibility, index_state)
workbench_definitions(id, project_id, image, setup_script, services_json, egress_allowlist_json, version)
workbenches(id, project_id, kind[project|mission|scratch], provider, provider_ref, state, snapshot_ref, created_at)
project_secrets(id, project_id, name, ciphertext, kms_key_ref, created_by, last_used_at)
conversations(id, user_id, project_id?, title, created_at, archived_at)
messages(id, conversation_id, parent_id?, branch_index, role, content_json, model_id, usage_json, created_at)
missions(id, project_id, created_by, objective, mode, autonomy, budget_json, state, base_ref, branch, pr_url, created_at, completed_at, outcome)
plan_steps(id, mission_id, seq, title, intent, state, risk, evidence_expected_json)
tool_calls(id, mission_id|conversation_id, step_id?, tool, args_hash, args_redacted_json, risk_class, decision, status, started_at, ended_at, output_artifact_id?)
checkpoints(id, mission_id, tool_call_id?, kind, payload_json, payload_hash, risk_class, state, requested_at, resolved_at, resolved_by, decision, note, expires_at)
grants(id, mission_id|project_id, pattern_json, granted_by, expires_at)
run_events(stream, seq, ts, type, actor, visibility, data_json, PRIMARY KEY(stream, seq))  -- partitioned by month
memory_entries(id, scope[user|project|org], owner_id, content, source_ref, pinned, state[active|proposed|stale|deleted], created_by, updated_at)
artifacts(id, kind, owner_id, project_id?, conversation_id?, mission_id?, version, mime, size, storage_key, provenance_json, retention_class, created_at)
usage_ledger(id, org_id?, user_id, project_id?, mission_id?, kind[model|workbench|search|storage], quantity, unit, cost_micros, model_id?, created_at)
api_keys(id, org_id, prefix, hash, scopes_json, created_by, expires_at, revoked_at, last_used_at)
audit_events(id, ts, actor_type, actor_id, action, resource_type, resource_id, decision, ip, device_id, details_json)  -- append-only
notifications(id, user_id, kind, ref, state, created_at, read_at)
model_registry(id, provider, model_id, capabilities_json, pricing_json, status, data_policy_json)
```

### 26.3 Data rules

- UUIDv7 ids (time-ordered) everywhere.
- Every tenant-owned table carries `org_id`/`owner_id` and is protected by RLS policies keyed on a session variable set by the repository layer.
- `run_events` is append-only, partitioned monthly, retained per plan (e.g., 90 days hot, then summarized; Mission Reports retained indefinitely until user deletes).
- PII minimization: tool-call args stored redacted; full args only in encrypted artifacts with short retention.
- Deletion: user/project deletion cascades with tombstones; object storage lifecycle deletes; vector rows deleted synchronously from retrieval.
- Backups: PITR on Postgres (7–35 days), quarterly restore drills; object storage versioning on critical buckets.
- Migrations: forward-only, expand/contract pattern, run by CI with a lock; zero-downtime.

---

## 27. Infrastructure

### 27.1 Services (deployables)

| Service | Responsibility | Runtime |
|---|---|---|
| **api** (modular monolith) | Auth session exchange, REST endpoints, SSE streams, webhooks (GitHub), policy, projects, conversations, missions CRUD, artifacts metadata, notifications | Kotlin/Ktor, Cloud Run (min instances ≥ 2; SSE-friendly timeouts) or GKE |
| **gateway** | ModelGateway: adapters, routing, fallbacks, usage metering | Kotlin/Ktor, autoscaled |
| **agent-worker** | Restate service handlers: mission virtual objects, agent loop, tool router, Scouts | Kotlin, GKE Autopilot (long-lived connections) |
| **restate** | Durable execution cluster | Restate (3 nodes), GKE with persistent disks, or Restate Cloud |
| **broker** | Credential Broker (GitHub App JWT, token minting, provider keys) | Kotlin, isolated project/network, minimal IAM |
| **egress** | Egress proxy per region for workbenches and fetch | Envoy-based (STABLE) with Xavro policy filter |
| **indexer** | Index orchestration, embedding batches, summaries | Kotlin workers on GKE; heavy parsing inside workbench indexing VMs |
| **parser** | Document parsing / malware scanning (network-less) | Sandboxed containers (gVisor) |
| **notifier** | FCM, email | Kotlin |
| **console** | Web Console | TypeScript web app |
| **workbench provider** | Managed microVM sandboxes (MVP) → self-hosted Firecracker pool (`xavro-workbenchd`, Go) | Provider API / bare-metal or nested-virt hosts |

Infrastructure services: Cloud SQL/AlloyDB Postgres (HA), Memorystore for Valkey (or self-managed Valkey), Cloud Storage, Secret Manager, Cloud KMS, Pub/Sub only if needed for webhooks buffering (else Postgres outbox), OTel Collector, Grafana stack, Langfuse, Sentry, PostHog.

### 27.2 Networking

- Public: HTTPS load balancer with Cloud Armor (WAF, rate limiting) in front of api and console.
- Private: services in a VPC; broker in a separate subnet with only api/agent-worker ingress; workbench provider reached via private connectivity where available; egress proxy is the only path out from workbenches.

### 27.3 CI/CD

- Monorepo with Gradle (Android + server + protocol), pnpm for console, Go modules for workbenchd.
- GitHub Actions: lint, unit tests, Android instrumented tests on Gradle Managed Devices, server integration tests with Testcontainers (Postgres, Valkey, Restate), contract tests for `xavro-protocol`, eval smoke suite on PRs touching agent code, full eval nightly.
- Supply chain: dependency review, SBOM, cosign signing, provenance attestations; deploy via OIDC workload identity (no static cloud keys).
- Environments: dev (per-PR preview for console/api where feasible), staging (prod-like, synthetic users), production. Progressive delivery: canary 5% → 25% → 100% with automated rollback on SLO burn.
- Android: internal track on every main merge, closed testing weekly, staged production rollouts (1% → 10% → 50% → 100%) with crash-rate gates; Play App Signing; in-app updates for critical fixes.

### 27.4 Environments and configuration

- OpenTofu for all infrastructure; per-environment state; policy checks (no public buckets, encryption required).
- Feature flags via OpenFeature SDK with a self-hosted provider (e.g., Flagd/Unleash) — remote config for the app with signed payloads.

---

## 28. Testing strategy

| Level | Scope | Tools | Gate |
|---|---|---|---|
| Unit | Domain logic, policy rules, routing, parsers, reducers | JUnit 5 / Kotest, kotlinx-coroutines-test, Turbine | PR |
| Policy tests | Exhaustive risk-class matrix, dynamic risk patterns, grants, autonomy levels | Table-driven Kotest + property tests | PR (100% of rules covered) |
| Integration (server) | API + Postgres + Valkey + Restate + fake-free provider adapters against recorded contracts | Testcontainers; provider contract tests against sandbox keys in nightly | PR (local), nightly (live providers) |
| Protocol contract | `xavro-protocol` schemas compatible across app/server versions | Schema compatibility checks, golden JSON | PR |
| Android UI | Screens in all window size classes, dark/light, font scales, RTL | Compose UI tests, screenshot tests (Roborazzi/Paparazzi-class), Gradle Managed Devices (phone, foldable, tablet), Espresso for system integrations | PR |
| Accessibility | TalkBack labels, contrast, touch targets | Compose a11y checks, Accessibility Test Framework | PR |
| Performance | Startup, jank, memory | Macrobenchmark, Baseline Profiles | Nightly with thresholds |
| End-to-end | Sign-in → project → GitHub test org → mission → PR in a sandbox org | Maestro/UI Automator against staging; dedicated GitHub test org and app | Pre-release |
| API | OpenAPI conformance, auth/scopes, rate limits | Schemathesis-class property testing | PR (Platform phase) |
| Agent evaluation | §24 suites | Inspect | Nightly; release gate |
| Security | SAST, dependency scanning, secret scanning, DAST on api/console, authz (IDOR) tests, sandbox escape tests, injection suite, pentest | CodeQL, OSV-Scanner, gitleaks, ZAP, custom | PR / release / quarterly |
| Sandbox | Isolation (cross-tenant FS/network), egress policy, resource limits, cleanup | Custom test suite run against each image/provider release | Release |
| GitHub integration | Webhook signature/dup handling, token scoping, permission verification, PR flows | Test GitHub org + recorded webhook fixtures | PR + nightly live |
| Load | SSE fan-out, concurrent missions, gateway throughput | k6 / Gatling | Before each phase launch |
| Failure / chaos | Provider outage, workbench crash, Restate node loss, DB failover, network partitions; idempotency of side effects on retry | Fault injection in staging | Monthly game days |

Rule: **no mocks for systems Xavro owns in integration tests**; external providers are faked only at the network boundary with recorded contracts, and every fake is matched by a live nightly test.

---

## 29. Prioritization

### 29.1 Capability classification

| Capability | Class |
|---|---|
| Passkey/Google sign-in, sessions, device management | CRITICAL |
| Ask with streaming, multi-model routing + fallback, attachments, markdown/code rendering | CRITICAL |
| Projects with memory (visible, editable) | CRITICAL |
| GitHub App connection, repo binding, issues/PR read | CRITICAL |
| Workbench (managed microVM), egress proxy, credential broker | CRITICAL |
| Missions: plan, execute, verify, Trace, Trace Rail, Checkpoints, Mission Report | CRITICAL |
| Native diff review, test evidence, draft PR creation | CRITICAL |
| Policy engine with risk classes + autonomy levels | CRITICAL |
| Push notifications (checkpoint, completion) | CRITICAL |
| Adaptive layouts (phone, foldable, tablet) | CRITICAL |
| Budgets and live cost meter | CRITICAL |
| Audit log, kill switches | CRITICAL |
| Observability baseline + eval harness | CRITICAL (internal) |
| Repo indexing (tree-sitter + embeddings + Repo Map) | HIGH |
| Interactive terminal in project workbench | HIGH |
| Research mode with cited reports | HIGH |
| Steering during missions, PR review-comment iterations | HIGH |
| Browser verification (Playwright) | HIGH |
| Library (artifacts) with search | HIGH |
| Offline cache and queued compose | HIGH |
| MCP client with allowlisted servers | MEDIUM |
| SCIP precise code navigation | MEDIUM |
| Code edit mode (WebView editor) | MEDIUM |
| BYOK | MEDIUM (HIGH for Professional) |
| Scouts (parallel read-only sub-agents) | MEDIUM |
| Web Console admin for orgs, teams, sharing | MEDIUM (Professional) |
| Developer API + Xavro MCP server | LOW → HIGH in Platform phase |
| Self-hosted Firecracker pool | LOW → HIGH at scale |
| Scheduled/automated missions | LOW |
| Interactive open-web browsing agent | EXPERIMENTAL |
| On-device Gemini Nano features | EXPERIMENTAL |
| A2A interoperability | EXPERIMENTAL |
| Deployment tools, iOS/desktop clients, marketplace, voice-first mode, enterprise BYO-cloud workbenches | FUTURE |

### 29.2 The smallest version that already feels exceptional (MVP)

**"Fix it from your phone."** A user signs in with a passkey, connects one GitHub repository, asks a question about the code and gets a cited answer, then delegates a bug fix as a mission. Xavro prepares a cloud workbench, shows a plan, executes it with a live Trace and Trace Rail, asks one clear checkpoint before pushing, runs the tests, and delivers a native diff review with test evidence and a Mission Report; one tap opens a draft PR. Everything is honest, fast, and works beautifully on a phone and a foldable.

That is exceptional because no product does that loop natively on Android with this level of visibility and control.

---

## 30. Product roadmap

Phases are defined by capability and exit criteria, not by calendar dates; each phase starts only when the previous phase's exit criteria are met.

### Phase 0 — Foundation

- **Capabilities:** monorepo, CI/CD, `xavro-protocol`, design system (Meridian tokens and core components), identity + sessions, api skeleton with policy module, ModelGateway with 3 providers, Restate cluster + MissionEngine port, managed workbench integration + egress proxy + credential broker, observability baseline, eval harness with first 50 tasks, threat model.
- **User value:** none externally; internal dogfooding of Ask.
- **Dependencies:** cloud accounts, GitHub App registration, provider contracts, workbench vendor selection (spike comparing ≥ 2 vendors on isolation, start latency, egress control, cost).
- **Risks:** workbench vendor limits; Restate fit — **engine-swap spike** (implement one mission type on both Restate and Temporal behind the port, compare).
- **Exit criteria:** an internal user can sign in on Android, chat with streaming and fallback, and an internal-only mission can edit a test repo in a workbench and push to a branch through the broker, with full traces.

### Phase 1 — MVP ("Fix it from your phone")

- **Capabilities:** everything CRITICAL in §29.1.
- **User value:** verified fixes and small features delivered as draft PRs from Android.
- **Dependencies:** Phase 0; eval suite at 150 tasks; security review + external pentest.
- **Risks:** agent quality on unfamiliar stacks; cost per mission; approval UX.
- **Success criteria:** ≥ 45% verified completion on Xavro-Missions; ≥ 60% acceptance of completed missions by beta users; P50 mission start < 10 s; crash-free sessions ≥ 99.5%; zero critical security findings open.

### Phase 2 — First usable product (public launch)

- **Capabilities:** repo indexing + Repo Map, interactive terminal, Research mode, steering, PR review iterations, browser verification, Library, offline cache, budgets UX refinement, Arabic + English localization, Play Store launch.
- **User value:** daily-driver workspace for mobile-heavy developers; research for non-developers.
- **Success criteria:** WVO ≥ 2 for weekly actives; 4-week retention ≥ 35% among users with one verified outcome; cost per verified outcome within plan margins.

### Phase 3 — Advanced product

- **Capabilities:** MCP client (allowlisted), SCIP navigation, edit mode, Scouts, BYOK, project-level grants, CI "drive to green", multi-repo projects, large-screen power features (three-pane mission, multi-window missions).
- **Success criteria:** verified completion ≥ 60%; steering-to-success ≥ 70%; unnecessary-checkpoint rate < 10%.

### Phase 4 — Professional product

- **Capabilities:** teams and shared projects, Web Console admin, roles, SSO, audit export, Cedar policies, sharing artifacts, self-hosted Firecracker pool (cost/residency), EU region.
- **Success criteria:** first paying teams; admin task completion without support; gross margin target met.

### Phase 5 — Flagship platform

- **Capabilities:** Developer API + Xavro MCP server, webhooks, SDKs, enterprise (SCIM, data residency, BYO-cloud workbenches, private model endpoints), scheduled missions, deployment tools with R3/R4 controls, A2A (if adoption justifies), iOS/desktop clients via shared KMP modules.
- **Success criteria:** third-party integrations live; enterprise security reviews passed; platform revenue share.

---

## 31. Competitive differentiation — explicit answers

- **What will Xavro do better?** Mobile control of real agent work: live Trace, steering, ground-truth checkpoints, native diff review and verified Mission Reports on Android — the best place to direct and own AI work away from a desk.
- **What will Xavro simplify?** Environment setup (auto-detected Workbench), approvals (risk classes + autonomy levels + scoped grants), and context (Projects that carry memory, repo knowledge and history across conversations and missions).
- **What will Xavro combine more elegantly?** Assistant, research, repository, terminal, agent and GitHub in one object model with one activity stream (Signals) and one permission system — instead of separate products glued by links.
- **What will Xavro make possible that is hard today?** Taking an issue from discovery to verified PR entirely from a phone; safely letting an agent work while you're away and responding to its one real question from a notification; understanding exactly what an agent did and why, with evidence.
- **Why should a developer choose Xavro?** It turns dead time into shipped work without sacrificing safety: credentials never enter the sandbox, every action is recorded, nothing irreversible happens without them, and results come with proof.
- **Why should a normal AI user choose Xavro?** A calm, premium assistant with excellent research reports and memory they can see and edit, organized into projects — without dark patterns.
- **Why should users keep returning?** Context compounds (projects get smarter), missions run while they're away and come back with verified results, and the cost is predictable.

---

## 32. Quality gate: the definition of "finished"

A feature is finished only when all of the following hold:

1. **States:** loading, empty, error (with specific cause), retry, offline, cancellation (where applicable), permission-denied, and unavailable (integration not configured) states are designed and implemented.
2. **Honesty:** no fake data, fake progress, fake success, placeholder content or simulated integrations in production builds; unavailable capabilities render an honest unavailable state with the reason and next step.
3. **Security:** authorization enforced server-side; policy tests added; threat-model delta reviewed; secrets never logged.
4. **Accessibility:** TalkBack labels, focus order, contrast, 200% font scale, touch targets, reduced motion.
5. **Adaptive:** verified on compact, medium and expanded widths, foldable postures, landscape, multi-window; RTL.
6. **Telemetry:** traces, metrics and product events (allowlisted, no content) emitted; dashboards updated.
7. **Testing:** unit + integration + UI/screenshot tests; agent-affecting changes pass the eval gate.
8. **Performance:** within budgets in §6.8 and service SLOs.
9. **Recovery:** behaviour on process death, network loss, server restart, and duplicate delivery verified (idempotent).
10. **Documentation:** user-facing help text, internal README/ADR updated.

---

## 33. Final architectural blueprint

### 33.1 System diagram

```
┌─────────────────────────────── ANDROID APP (Kotlin, Compose) ───────────────────────────────┐
│ UI (Meridian) · Navigation 3 + M3 Adaptive · ViewModels · Room cache · Credential Manager     │
│ Signal client (SSE) · Terminal WS · FCM · CodeEditor (local WebView) · xavro-protocol (KMP)  │
└───────────────┬───────────────────────────────┬─────────────────────────────┬────────────────┘
                │ REST/JSON + SSE                │ WebSocket (PTY)              │ FCM (ids only)
┌───────────────▼───────────────────────────────▼─────────────────────────────┴────────────────┐
│                                    API (modular monolith)                                     │
│ AuthN (session exchange w/ Zitadel OIDC) · AuthZ + RLS · Policy · Projects · Conversations ·  │
│ Missions API · Checkpoints · Artifacts · Memory · Signals (SSE from run_events + Valkey) ·    │
│ GitHub webhooks · Notifications                                                               │
└──────┬───────────────┬──────────────────┬──────────────────┬────────────────┬────────────────┘
       │               │                  │                  │                │
┌──────▼─────┐  ┌──────▼──────┐   ┌───────▼────────┐  ┌──────▼──────┐  ┌──────▼───────┐
│  Gateway   │  │ Restate     │   │ Agent workers  │  │  Indexer    │  │  Broker      │
│ (models,   │◀─│ (durable    │◀─▶│ Mission VOs,   │  │ (tree-sitter│  │ (GitHub App  │
│ routing,   │  │ execution)  │   │ agent loop,    │  │ SCIP, embed)│  │ keys, tokens,│
│ fallback,  │  └─────────────┘   │ Tool Router →  │  └──────┬──────┘  │ provider/BYOK│
│ metering)  │                    │ Policy         │         │         │ keys)        │
└──────┬─────┘                    └───┬───────┬────┘         │         └──────┬───────┘
       │                              │       │              │                │
       ▼                              │       ▼              ▼                ▼
 Model providers                      │   MCP proxy ──▶ allowlisted MCP servers     GitHub API
 (Anthropic, OpenAI,                  │
  Google, OpenAI-compatible)          ▼
                              ┌────────────────────── WORKBENCH LAYER ──────────────────────┐
                              │ microVM per workbench (managed → self-hosted Firecracker)   │
                              │ xavro-agentd: PTY, processes, fs ops, git, tests, Playwright │
                              │ No credentials · default-deny egress ──▶ Egress proxy        │
                              │   (registries allowlist, GitHub via broker-injected token)   │
                              └──────────────────────────────────────────────────────────────┘

Data: Postgres (OLTP, RLS) · Postgres+pgvector (vectors) · Object storage · Valkey · Restate journals
Observability: OTel → Collector → Tempo/Loki/Mimir + Langfuse; Sentry; PostHog
Admin: Web Console (TypeScript) → same API + internal admin endpoints (WebAuthn, JIT, audited)
```

### 33.2 Boundaries and responsibilities

| Boundary | Trust | Responsibility | Must never |
|---|---|---|---|
| Android app | Untrusted client | Render, collect intent, collect decisions, cache | Enforce security alone; hold provider/GitHub tokens; run agent logic |
| API | Trusted | Authenticate, authorize, persist, stream, enforce policy for user actions | Call providers directly (goes through gateway); hold GitHub App key |
| Gateway | Trusted | Model access, routing, metering | Execute tools |
| Agent workers | Trusted code, **untrusted reasoning** | Run loop, route tools through Policy | Bypass Tool Router; hold long-lived credentials |
| Policy | Trusted | Decide allow/checkpoint/deny | Depend on model output for the decision itself |
| Broker | Highest trust | Mint/inject scoped credentials | Expose raw keys to any other component |
| Workbench | Untrusted | Execute code | Hold credentials; reach arbitrary network |
| Egress proxy | Trusted | Enforce network policy; inject GitHub auth | Allow private ranges/metadata |
| MCP servers | Untrusted (third-party) | Provide tools | Be called outside the proxy |

### 33.3 Information flow: a mission, end to end

1. App → `POST /missions` (objective, project, mode, autonomy, budget). API authorizes (user owns project, has push access via GitHub user token check), persists mission, starts Mission virtual object in Restate.
2. Agent worker: intake → (clarify via checkpoint) → requests workbench (fork project snapshot) → loads Repo Map + memory + retrieval → planning call via Gateway (Reasoning role) → Plan persisted, `plan.updated` Signal.
3. Signals are appended to `run_events` and published to Valkey; the app's SSE stream receives them; FCM sent if the user is away and a checkpoint is needed.
4. Execution loop: model (Coding role) proposes tool call → Tool Router validates schema → Policy decides → allowed calls execute (workbench via agentd; GitHub via broker; web via fetch proxy) → outputs redacted, wrapped as untrusted, summarized → next iteration. Each step is a durable Restate step with an idempotency key.
5. Checkpoint: Restate awakeable created; API exposes it; user decides on the app; decision (signed, payload-hashed) resolves the awakeable; execution continues.
6. Verification: tests run; results parsed; self-review; Report generated (Fast/Reasoning roles), stored as artifact.
7. Handoff: push branch through egress (broker-injected token scoped to the repo, contents:write); draft PR via broker-signed API call; `github.pr.opened` Signal; notification.
8. Telemetry: one trace spans all of the above; usage ledger updated per model call and workbench minute; audit events for every R2+ action and every decision.

---

## 34. Dangerous assumptions, hidden complexity and missing requirements

| Item | Why it matters | Resolution |
|---|---|---|
| "Mobile users will tolerate long mission latency" | They check briefly and leave | Notifications at meaningful moments; Trace Rail communicates progress honestly; Reports are skimmable. |
| "Approvals keep users safe" | Fatigue → rubber-stamping (ASI09) | Risk-based checkpoints, ground-truth payloads, isolation as primary control. |
| "Indexing is a one-time cost" | Repos change constantly | Incremental indexing by blob SHA; lazy summaries. |
| "Sandboxes are cheap" | Idle VMs burn money | Aggressive snapshot/pause; warm pools sized by forecast; per-plan limits. |
| "One model does everything" | Quality/cost/outage risk | Role-based routing with eval-driven selection. |
| Hidden complexity: **cost accounting** | Users must see cost before/during work | Usage ledger from day one; estimates from historical similar missions. |
| Hidden complexity: **idempotent side effects** | Retries could double-post PRs/comments | Idempotency keys on every external write; check-before-create (find PR by head branch). |
| Hidden complexity: **provider message format drift** | Tool-call/reasoning formats differ | Normalized internal format in gateway; adapter contract tests. |
| Hidden complexity: **localization incl. RTL** | Arabic-speaking initial market likely | RTL and Arabic in Phase 2 exit criteria; code/terminal stay LTR. |
| Missing requirement: **legal/compliance** | Privacy policy, DPA, data retention, export (GDPR/regional laws), Play data safety form, AI disclosure obligations (e.g., EU AI Act transparency) | Add a compliance workstream in Phase 1. |
| Missing requirement: **billing & plans** | Cost-heavy product needs metering + plans | Google Play Billing for in-app subscriptions (required for digital goods on Android) + web billing for teams; usage-based add-ons. |
| Missing requirement: **abuse prevention** | Free compute attracts mining, spam PRs, phishing pages | Play Integrity signals, account age limits on workbench resources, egress allowlist, rate limits on GitHub writes, anomaly detection. |
| Missing requirement: **support tooling** | Users will ask "what did the agent do?" | Mission replay for support with user-granted access. |
| Missing requirement: **content safety** | Assistant misuse | Provider safety + Xavro usage policy + abuse reporting. |
| Missing requirement: **data export** | Trust and portability | Export conversations, memory, reports (JSON/Markdown) from Settings. |
| Vendor risk: managed workbench provider | Pricing, outages, policy changes | `WorkbenchProvider` port; second vendor qualified; self-host path in Phase 4. |
| Vendor risk: Restate | Younger project | `MissionEngine` port; Temporal spike in Phase 0. |
| Platform risk: Google Play policy for AI apps and code execution | Apps executing downloaded code locally are restricted | Xavro executes nothing locally; all execution is server-side, which is compliant; keep AI-generated content reporting features required by Play policy. |

---

## 35. Research sources

Verified October 2026 unless marked. Maturity labels reflect status at that date.

**Protocols**
- MCP 2026-07-28 specification overview and changes — [MCP blog: protocol tag](https://blog.modelcontextprotocol.io/tags/protocol/) (STABLE core; new features PREVIEW in ecosystem); [Breaking changes summary](https://www.developersdigest.tech/blog/mcp-2026-07-28-breaking-changes); [Governance commentary](https://pub.towardsai.net/model-context-protocols-2026-07-28-spec-puts-governance-before-speed-2b240295a1c5).
- A2A — [Linux Foundation press release: A2A surpasses 150 organizations](https://linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) (EXPERIMENTAL for Xavro).

**Security**
- [OWASP Top 10 for Agentic Applications for 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026); item summaries via [Cycode](https://cycode.com/blog/owasp-top-10-agentic-applications/).
- GitHub Apps — [Best practices for creating a GitHub App](https://docs.github.com/en/enterprise-server@3.16/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app); [GitHub Apps cheat sheet](https://github.com/github-developer/github-apps-cheat-sheet) (STABLE).

**Execution and durability**
- [Fly.io: Firecracker vs gVisor](https://fly.io/learn/firecracker-vs-gvisor/) (vendor-authored; STABLE technologies).
- Sandbox landscape: [E2B/Daytona/Firecracker overview](https://www.spheron.network/blog/ai-agent-code-execution-sandbox-e2b-daytona-firecracker/), [Beam: E2B alternatives](https://www.beam.cloud/blog/best-e2b-alternatives) (vendor/secondary; verify per vendor).
- [Restate vs Temporal](https://restate.dev/vs/temporal) (vendor-authored, claims unverified); [DBOS vs Temporal](https://toolradar.com/compare/dbos-vs-temporal) (secondary).

**Android**
- [Compose Material 3 Adaptive releases](https://developer.android.com/jetpack/androidx/releases/compose-material3-adaptive) (1.3.0 STABLE, 1.4.0-alpha PREVIEW).
- [Jetpack Navigation 3 is stable](https://developer.android.com/blog/posts/jetpack-navigation-3-is-stable) (STABLE).
- [Orientation and resizability changes in Android 16](https://android-developers.googleblog.com/2025/01/orientation-and-resizability-changes-in-android-16.html) (STABLE platform behaviour).
- [ML Kit GenAI Prompt API](https://developer.android.com/blog/posts/ml-kit-s-prompt-api-unlock-custom-on-device-gemini-nano-experiences) and [alpha release post](https://android-developers.googleblog.com/2025/10/ml-kit-genai-prompt-api-alpha-release.html) (PREVIEW).
- [Android identity / Credential Manager](https://developer.android.com/identity) (STABLE).

**Observability**
- [OpenTelemetry GenAI semantic conventions repository](https://github.com/open-telemetry/semantic-conventions-genai) (development status); [GenAI agent spans (moved notice)](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans).

**Market**
- [AI coding agents compared (2026)](https://ssojet.com/blog/ai-coding-agents-compared) (secondary; benchmark numbers vendor-reported, UNPROVEN independently).

**Data**
- [Vector database comparison 2026](https://alexcloudstar.com/blog/vector-database-comparison-2026/), [pgvector vs Qdrant](https://encore.dev/articles/pgvector-vs-qdrant) (secondary).

**Browser automation**
- [Playwright MCP notes and cost comparison](https://morphllm.com/playwright-mcp) (secondary; Playwright itself STABLE).

Items to re-verify at implementation start: MCP 2026-07-28 SDK support in the Kotlin SDK; NavigationSuiteScaffold API changes in M3 Adaptive 1.4; OTel GenAI convention version; workbench vendor isolation and pricing; Restate Kotlin SDK version; GitHub App permission names; Google Play policies for AI and subscriptions.
