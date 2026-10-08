# PHASE B — XAVRO AI: MASTER BUILD PROMPT

> This prompt is self-contained. It is the single source of truth for building Xavro AI. It was derived from `PHASE_A_MASTER_BLUEPRINT.md`; where this prompt and any other document disagree, this prompt wins. Where a decision here turns out to be technically impossible, stop, record the conflict in `docs/adr/`, and propose the smallest change that preserves the intent. Do not silently deviate.

---

## 0. Your role and mission

You are the engineering team building **Xavro AI**, a premium Android AI workspace. You will build it in phases (§29), starting with Phase 0 (Foundation) and Phase 1 (MVP). You write production code, tests, infrastructure-as-code and documentation. You do not write demos, mockups or simulated features.

**Product in one sentence:** Xavro lets a user think with AI and then delegate real engineering and research work to an agent that runs in a secure cloud Workbench, while the user directs, observes, approves and reviews everything from a native Android app.

**Core philosophy the product must embody:** THINK → UNDERSTAND → PLAN → BUILD → EXECUTE → VERIFY → COLLABORATE. Every mission visibly passes through these stages.

### 0.1 Non-negotiable rules

1. **No fake features.** Never ship fake integrations, fake repository data, fake terminal output, fake model responses, fake progress, fake success states, fake deployments, or placeholder functionality presented as real. If a capability cannot work because credentials or infrastructure are missing, implement the real integration boundary (interface, adapter, configuration) and render an **honest unavailable state** that names what is missing and how to enable it. Seed/sample data is allowed only in tests and in clearly labelled developer builds.
2. **Security is enforced by trusted backend components**, never only by the UI and never by instructions to the model. The AI must not be able to bypass authentication, authorization, repository permissions, tool permissions, execution isolation, or approval requirements.
3. **The phone is the control surface; the cloud is the workbench.** The Android app never executes agent work, builds, tests or user code locally.
4. **Never expose private chain-of-thought.** Expose actions, observations, plans, decisions and evidence.
5. **Every external dependency sits behind a Xavro-owned interface (port).**
6. **A feature is not done until it meets the Definition of Done (§28).**
7. **Do not merge to `main` without human review.** Agents working on this repository open pull requests; humans merge.

---

## 1. Product identity

- **Name:** Xavro AI. **Design language:** Meridian. **Signature UI element:** the Trace Rail.
- **Personality:** calm, precise, instrument-grade, trustworthy, developer-friendly. Not playful, not neon, no mascots, no sparkles, no glowing orbs.
- **Primary users at launch:** mobile-heavy developers, professional developers away from their desk, maintainers/tech leads, and AI power users (assistant + research features).
- **North Star:** Weekly Verified Outcomes per active user (missions or research tasks ending in an accepted result: merged/approved PR, applied change, saved/shared report, with no rollback within 7 days). Instrument it from Phase 1.

### 1.1 Vocabulary (use these exact terms in UI, code and docs)

| Term | Definition |
|---|---|
| **Ask** | The conversational surface and destination. |
| **Project** | Context container: repository, files, conversations, missions, memory, workbench definition, settings. |
| **Mission** | A delegated multi-step job executed by the agent, with a Plan, Trace, Checkpoints, Changes and a Mission Report. |
| **Plan** | Ordered, editable mission steps. |
| **Trace** | The user-safe, truthful activity stream of a mission. |
| **Trace Rail** | Compact segmented progress element representing plan steps and live state. |
| **Checkpoint** | A point where a mission needs a user decision. |
| **Workbench** | An isolated cloud microVM where a project's code runs. |
| **Mission Report** | Structured outcome document every mission ends with. |
| **Library** | All artifacts and files across the user's projects and conversations. |
| **Memory** | What Xavro remembers; always visible and editable. |
| **Signal** | Internal name for a real-time event (not user-facing). |

Do not introduce a separate "Tasks" area, an agent marketplace, or social features.

---

## 2. Locked architecture decisions

| Area | Decision | Port / boundary |
|---|---|---|
| Android | Kotlin, Jetpack Compose, Material 3 + Meridian token layer, Compose Material 3 Adaptive **1.3.x**, Navigation 3, Hilt, Coroutines/Flow, kotlinx.serialization, OkHttp (HTTP/2, SSE, WebSocket), Room, DataStore, WorkManager, Coil, Credential Manager, FCM, Baseline Profiles. minSdk 28, target/compile latest stable SDK (≥ 36). | `XNavigationSuite` wraps `NavigationSuiteScaffold`; `WindowInfo` wrapper isolates window-size APIs. |
| Code viewing | Native Compose viewer/diff with server-computed tree-sitter highlight tokens. | `HighlightService` (server). |
| Code editing | CodeMirror 6 bundled locally in an isolated WebView (no network, `WebMessagePort` bridge). | `CodeEditor` interface. |
| Terminal rendering | Native Compose terminal renderer over a proven VT emulator state machine (verify licence, e.g. Termux `terminal-emulator`, Apache-2.0). PTY runs in the Workbench. | `TerminalSurface` interface. |
| Markdown | JetBrains `markdown` parser → native Compose renderer; math and Mermaid rendered server-side to SVG. | `RichContentRenderer`. |
| Backend | Kotlin on JVM 21, Ktor server. Modular monolith `api` + separate deployables `gateway`, `agent-worker`, `indexer`, `broker`, `notifier`, `parser`. | Module boundaries enforced by Gradle. |
| Shared contract | `xavro-protocol` Kotlin Multiplatform module: DTOs, Signal schema, tool schemas, error codes. Used by Android and server. | Versioned, compatibility-tested. |
| Durable execution | **Restate** (Kotlin SDK). Missions are Virtual Objects keyed by mission id; checkpoints are awakeables. Temporal is the qualified fallback. | `MissionEngine` port; engine APIs confined to one module. |
| Agent loop | In-house. No third-party agent framework in the runtime. | `AgentLoop`, `ToolRouter`, `Policy`. |
| Models | In-house `ModelGateway` with adapters: Anthropic, OpenAI, Google, OpenAI-compatible, optional OpenRouter. Registry is data. | `ModelProvider` adapter interface. |
| Workbench | Managed microVM sandbox provider (hardware-virtualized isolation required; plain containers forbidden) at MVP; self-hosted Firecracker (`xavro-workbenchd`, Go) in Phase 4. | `WorkbenchProvider` port. |
| In-VM supervisor | `xavro-agentd` (Go, static binary) baked into Workbench images: PTY, processes, fs ops, git, test runners, Playwright control. | gRPC/HTTP over the provider's control channel. |
| Network from Workbench | Default-deny egress via Envoy-based egress proxy with Xavro policy filter; GitHub auth injected by proxy for the bound repo only. | `EgressPolicy`. |
| Credentials | Credential Broker is the sole holder of GitHub App private key, provider API keys and BYOK keys; mints short-lived scoped tokens. | `CredentialBroker`. |
| Databases | PostgreSQL (OLTP, Row-Level Security), separate PostgreSQL + pgvector (HNSW) for vectors, S3-compatible object storage, Valkey (cache, rate limits, pub/sub). | `VectorStore`, `BlobStore`. |
| Identity | OIDC via Zitadel (managed). Android: Credential Manager (passkeys first, Sign in with Google, email magic link fallback). Xavro API issues its own access tokens (≤ 15 min) and rotating, device-bound refresh tokens. GitHub is a linked account only. | `IdentityProvider`. |
| Code intelligence | tree-sitter (parse/chunk/outline/highlight), SCIP indexers where buildable, ripgrep in Workbench, embeddings via gateway, hybrid retrieval with reciprocal-rank fusion. Keys: repo + blob SHA. | `CodeIndex`. |
| Browser | Playwright + Chromium inside the mission Workbench. | `browser.*` tools. |
| Web search | Two search providers behind a port, chosen by Xavro's research eval. | `SearchProvider`. |
| Policy | Typed Kotlin rules in a `policy` module; Cedar in Phase 4 for org policies. | `Policy.decide()`. |
| Push | FCM data messages with ids only. | `PushSender`. |
| Observability | OpenTelemetry everywhere; Collector → Tempo/Loki/Mimir; Langfuse for LLM/agent traces and eval datasets; Sentry for crashes/exceptions; PostHog for allowlisted product events. GenAI semantic conventions pinned in one `telemetry-genai` module. | `Telemetry`. |
| Evaluation | UK AISI Inspect harness + Xavro suites. | `evals/` |
| Cloud | Google Cloud at launch (Cloud SQL/AlloyDB, Cloud Run/GKE Autopilot, Cloud Storage, Secret Manager, KMS, Cloud Armor). OpenTofu for all infra. No proprietary DB features. | Portability rules. |
| Web Console | TypeScript web app for admin, billing, API keys, docs, optional desktop review. | Uses public + internal APIs. |
| CI/CD | GitHub Actions with OIDC to cloud, cosign signing, SBOMs, Gradle Managed Devices, staged Play rollouts. | — |

Record each decision as an ADR in `docs/adr/NNNN-title.md` during Phase 0.

---

## 3. Repository layout

```
/android/                     Android app (Gradle, Kotlin)
  app/                        Application module, DI graph, navigation host
  core/designsystem/          Meridian tokens, theme, components, icons
  core/ui/                    Shared UI (Trace Rail, Signal list, diff, code view, terminal surface, markdown)
  core/data/                  Repositories, Room, DataStore, network clients, Signal client
  core/auth/                  Credential Manager, session, token storage
  core/telemetry/             OTel/Sentry/PostHog wrappers
  feature/ask/  feature/projects/  feature/missions/  feature/library/
  feature/settings/  feature/onboarding/  feature/inbox/  feature/search/
  benchmark/  baselineprofile/
/protocol/                    xavro-protocol (KMP): DTOs, Signals, tool schemas, errors
/server/
  api/  gateway/  agent-worker/  indexer/  broker/  notifier/  parser/
  modules/policy/  modules/domain/  modules/persistence/  modules/github/
  modules/workbench/  modules/mission-engine/  modules/tools/  modules/memory/
  modules/retrieval/  modules/telemetry-genai/  modules/search/
/workbench/
  agentd/                     Go in-VM supervisor
  images/                     Workbench image definitions (base + toolchains)
  workbenchd/                 (Phase 4) Firecracker pool manager
/console/                     Web Console (TypeScript)
/evals/                       Inspect tasks, fixtures, pinned repos manifest, injection suite
/infra/                       OpenTofu modules and environments (dev, staging, prod)
/docs/                        adr/, xavro/ (blueprint + this prompt), runbooks/, threat-model/
```

Root-level `AGENTS.md` must describe build/test commands and conventions for coding agents.

---

## 4. Navigation and complete screen inventory

### 4.1 Global navigation

- Destinations: **Ask · Projects · Missions · Library**.
- Compact width (< 600 dp): bottom navigation bar. Medium (600–839 dp): navigation rail with a compose FAB at the top. Expanded and above (≥ 840 dp): navigation rail (permanent drawer at ≥ 1600 dp) plus list-detail panes and optional supporting pane.
- Top app bar on every destination: global search, **Inbox** (badge with count of waiting checkpoints and unseen completions), avatar → Profile & Settings.
- A **"Needs you" strip** appears at the top of Ask when any checkpoint is waiting or a mission finished since the last visit.
- Predictive back on all screens; back closes the innermost pane first.

### 4.2 Screens (every screen must implement loading, empty, error, offline, and permission/unavailable states)

| # | Screen | Required contents and behaviour |
|---|---|---|
| S1 | Welcome | Three statements (Ask anything · Delegate real work · Stay in control); primary button "Continue" (Credential Manager: passkey or Google); secondary "Other sign-in options" (email magic link). Offline: inline "You're offline" + retry. No carousel. |
| S2 | Personalize (skippable) | Two questions: primary use (Coding / Research & writing / Both); response style (Concise / Detailed). Answers become visible personal-memory entries. |
| S3 | Ask — conversation | Message list; composer; mode chip (Ask / Research / Mission); model chip (Auto or model); context tray; activity chips; citations pane; message actions (copy, share, regenerate, branch navigation, edit, save to project, run as mission). |
| S4 | Ask — history | List of conversations (search, filters: project, date); swipe to archive with undo. |
| S5 | Projects — list | Project cards: name, repo, last activity, running missions (Trace Rails), waiting checkpoints. Empty state explains projects with "New project". |
| S6 | New project | Name, description, add repository (GitHub) or skip. GitHub connection flow via Custom Tab to App installation; App Link return; repository picker listing only repos the installation covers AND the user can push to. |
| S7 | Project — Overview | Repo Map summary, detected commands (editable, confirmable), index status with real progress, open checkpoints, recent missions, suggested missions from open issues, honest unavailable states. |
| S8 | Project — Conversations | Project-scoped conversations. |
| S9 | Project — Missions | Missions in this project, filter by state. |
| S10 | Project — Code | File tree, native code viewer with breadcrumbs, search (text, symbols, semantic), go-to-definition/find-references when SCIP exists (fallback labelled), Edit (CodeMirror component). |
| S11 | Project — GitHub | Branches, pull requests (list + detail with checks, review comments), issues (list + detail with "Start mission from issue"). |
| S12 | Project — Files | Uploads and generated files; drag-and-drop on large screens. |
| S13 | Project — Memory | Entries with source, pinned toggle, edit, delete, proposals awaiting confirmation, stale flags. |
| S14 | Project — Workbench | Environment definition (image, setup script, services, egress allowlist, secret references), state, start/stop, interactive Terminal, background processes. |
| S15 | Project — Settings | Default models, autonomy default, budget default, enabled tools, MCP servers (Phase 3), permissions/grants, danger zone (archive/delete). |
| S16 | Mission Composer | Objective (text, voice, attachments), project/repo/base branch, mode (Plan first / Go), autonomy (Careful / Balanced / Autonomous), budget (estimate range shown, user cap), Start. |
| S17 | Mission | Header with title, state label, Trace Rail, cost meter, Pause/Resume/Cancel, Steer composer. Tabs/panes: **Plan**, **Trace**, **Changes**, **Terminal**, **Browser**, **Files**, **Report**. Compact: tabs. Medium: two panes. Expanded: three panes (Plan · Trace · selectable third). |
| S18 | Checkpoint | Sheet (compact/medium) or side dialog (expanded): action in plain words, exact payload rendered by trusted code, agent's explanation (labelled), risk class with "Why is this risky?", choices: Approve · Approve similar for this mission · Edit (where meaningful) · Decline with note. Not dismissible by outside tap for R3+. |
| S19 | Diff review | File list grouped (source/tests/config/generated) with +/- and one-line rationales; unified diff (compact) or split (≥ 840 dp); collapse unchanged; wrap toggle; long-press line → comment to agent; "Explain this file's changes". |
| S20 | Mission Report | Outcome, status badge (Verified / Unverified / Partial / Failed / Blocked), changes by group, evidence (commands, exit codes, test summaries, screenshots), decisions & assumptions, risks & follow-ups, cost & duration, links. Every claim links to its Trace event. Actions: Open draft PR / Update PR, Start follow-up mission, Share (Phase 4). |
| S21 | Missions — list | All missions grouped: Needs you · Running · Recent · Failed/Blocked. Live Trace Rails. Long-press quick actions. |
| S22 | Research report | Scope card, Research Trace, report with citations and claim confidence, bibliography with access dates. |
| S23 | Library | Artifacts across everything; filters (kind, project, source, date); full-text search; viewers per kind. |
| S24 | Inbox | Checkpoints, completions, failures, mentions (Phase 4); swipe to mark read. |
| S25 | Global search | Conversations, missions, artifacts, projects, memory; recent searches. |
| S26 | Settings | Account; Personal memory; Models (preference Faster/Balanced/Best, providers, BYOK Phase 3); Notifications (per channel); Appearance (theme, dynamic colour opt-in, true black, font); Language (English, Arabic + RTL in Phase 2); Integrations (GitHub link, installations, MCP in Phase 3); Usage (spend, limits); Security (passkeys, devices, sessions, agent activity log, secure-screen toggle); Data (export, delete account); About/legal. |
| S27 | Terminal (full screen) | PTY session, extra-keys row (Esc, Tab, Ctrl, Alt, arrows, `|`, `~`, `/`, `-`), hardware keyboard passthrough, selection/copy, pinch zoom, scrollback 10,000 lines, reconnect handling. |

---

## 5. Meridian design system — implement exactly

### 5.1 Tokens

Implement as Kotlin objects in `core/designsystem` and as CSS variables in `console/`.

**Neutral (Graphite):** `surface.base` L #FAFAF8 / D #0F1012 · `surface.raised` #FFFFFF / #16171A · `surface.sunken` #F1F1EE / #0B0C0D · `surface.overlay` #FFFFFF / #1C1D21 · `outline.hairline` #E3E2DE / #2A2B30 · `outline.strong` #C9C8C2 / #3A3B42 · `text.primary` #17181A / #ECECEA · `text.secondary` #55565B / #A7A8AD · `text.tertiary` #7C7D82 / #7D7E85.

**Accent (Verdigris):** `accent.primary` #1F6F66 / #5CC2B3 · `accent.onPrimary` #FFFFFF / #00201C · `accent.container` #D3EEE9 / #123A35.

**State:** `state.active` = accent · `state.attention` #B26A00 / #F2B35B · `state.success` #2E7D32 / #7BC67E · `state.danger` #B3261E / #F2B8B5 · `state.info` #3B5BA9 / #A9BFF2. **Diff:** added bg/fg #E6F4EA/#1B5E20 (L), #13261A/#9FD8A4 (D); removed bg/fg #FCE8E6/#8C1D18 (L), #2C1513/#F2B8B5 (D).

Validate every text/background pair for WCAG 2.2 AA in a unit test. Dynamic colour is an opt-in setting replacing only the accent family; state colours never change. Optional "True black" dark variant.

### 5.2 Type, shape, spacing, icons, motion

- Fonts: IBM Plex Sans (UI), IBM Plex Mono (code/data/terminal/numbers), IBM Plex Sans Arabic + Noto fallbacks. Scale (sp): Display 34/40, Headline 24/30, Title 18/24, Body 15/22, Body-small 13/18, Label 12/16, Code 13/20 (compact) 14/21 (expanded). Tabular figures for numbers. Support 200% font scale with reflow.
- Radii: containers 16 dp; sheets 28 dp top; chips 8 dp; inputs 12 dp; code blocks 10 dp. Elevation by tone + 1 dp hairlines; shadows only on floating elements.
- **Meridian notch:** 2 dp leading-edge accent line on mission-related containers carrying the Trace Rail state colour.
- Spacing: 4 dp base; gutters 16/24/32 dp by width class; prose max width ~680 dp; touch targets ≥ 48 dp.
- Icons: Material Symbols Rounded weight 300 + custom Xavro set (Mission, Checkpoint, Workbench, Trace, Plan, Verified, Unverified, Repo Map) on a 24 dp grid, 1.75 dp stroke.
- Motion: Material 3 motion scheme; 150 ms micro, 250 ms component, 350 ms container transform; shared element Mission card → Mission header; no per-token text animation; honour "Remove animations".
- Haptics: tick on send, confirm on approve, single heavy on destructive confirm, none during streaming.

### 5.3 Trace Rail specification

Equal-width segments per plan step (never time-weighted). Completed: solid accent. Running: slow constant shimmer (no fake percentage). Waiting on checkpoint: amber, single pulse every 2 s. Failed: danger colour with notch. Pending: hairline outline. Height 4 dp in cards, 6 dp in Mission header. Reduced motion: static. TalkBack: "Step 3 of 7, running: Run tests".

### 5.4 Component rules

Composer grows to 6 lines; attachments as chips; voice morphs into send. User messages in tonal containers (trailing side, mirrored in RTL); assistant messages full width, no bubble. Activity chips: icon + verb + object + duration; tap to expand inputs/outputs. Mission card: notch, title, project, Trace Rail, state, cost, "Needs you" badge. Snackbars only for undoable actions. Inline "Copied" label (no toast).

### 5.5 Accessibility

TalkBack labels on all elements; Trace announcements polite and throttled (≤ 1 per 5 s); Checkpoints assertive; streaming read by completed paragraph; colour never sole carrier; keyboard and switch access; defined focus order per pane layout; RTL mirroring with code, diff and terminal kept LTR.

---

## 6. Adaptive Android behaviour

- Use window size classes (width) compact/medium/expanded/large/extra-large via the `WindowInfo` wrapper.
- Implement layouts in §4.2 S17 and the table: Ask (single → list-detail → list-detail + supporting pane), Projects (list → list-detail), Mission (tabs → two panes → three panes), Diff (unified → split ≥ 840 dp), Checkpoint (sheet → side dialog).
- Foldables: tabletop posture puts live content top, controls bottom; book posture never places content across the hinge. Folding/unfolding preserves scroll, composer text and selected pane.
- No orientation locks. Fully resizable. Desktop windowing supported.
- Drag-and-drop in and out (files into composer/Files; code blocks/artifacts out).
- Hardware keyboard shortcuts: Ctrl+Enter send, Ctrl+K search, Ctrl+N new mission, Ctrl+/ shortcut help, Esc stop streaming; hover states; right-click menus; mouse wheel in diff/terminal.
- Open a Mission in an adjacent window on large screens.
- Background: no local agent work; FCM; WorkManager for uploads/sync; foreground service only while actively streaming a visible terminal; progress-style ongoing notification for running missions (user-controllable channel).
- Offline: Room cache of conversations, reports, recent artifacts, project metadata; queued messages labelled "Will send when online"; queued missions stay drafts and require confirmation after reconnect; offline banner.
- Performance budgets (CI-enforced via Macrobenchmark): cold start to interactive Ask < 1.2 s P50 / < 2.0 s P90 on a mid-range 2024 device; < 1% janky frames on long lists and diffs; 60 fps streaming at 100 tokens/s; download size < 25 MB; < 300 MB memory with 10k-line terminal + 5k-line diff open.

---

## 7. Ask (assistant) — requirements

1. Stream responses over SSE with resumable event ids. If the app backgrounds, the server completes the response; the app shows it on return.
2. First visible feedback within 300 ms of send. If the first token takes > 2 s, show "Thinking · <model name>"; after 10 s show elapsed time. Stop is always visible while streaming; partial output is preserved and marked "Stopped".
3. Message editing creates a branch; regenerate (same or another model) creates a branch; branches navigable ("< 2/3 >").
4. On provider failure, fall back automatically per routing policy and show "Answered by <model> (fallback)". If all candidates fail, show the specific error with Retry and Try another model.
5. "Continue" appears when output was truncated by limits and continues seamlessly.
6. Citations: inline numbered markers → Sources pane with title, domain, retrieval time, quoted passage.
7. Code blocks: server-computed highlighting, copy, wrap toggle, "Apply to project" (creates a mission proposal or direct edit proposal).
8. Tables (scrollable), math and Mermaid (server-rendered SVG).
9. Inputs: text, voice (on-device recognition where available; server transcription only with consent), camera/images, any file (server-parsed in sandboxed parser), links (fetched preview), GitHub URLs (resolved via the GitHub App with permission checks), shared text from other apps.
10. Context tray lists exactly what will be sent with toggles; token usage indicator turns amber at 80% of the model window.
11. Ask tools allowed: `web.search`, `web.fetch`, project read tools (`fs.read`, `search.*`, `project.explain`) when a project is attached, and a **scratch Workbench** code runner (no project data, no secrets, strict egress). **Any state-changing action is forbidden in Ask**; the assistant must offer a mission proposal card instead.
12. Mission proposal card: drafted objective, inferred project/repo, suggested mode; accepting creates a mission linked to the conversation; mission milestones post compact updates back into the conversation.

---

## 8. Projects and project intelligence — requirements

1. Project creation with or without a repository. One primary repository per project at MVP; multi-repo in Phase 3.
2. On repository binding, run **Workbench preparation**: clone default branch → detect languages, frameworks, package managers, build/test/lint commands (verify each by dry run) → read instruction files (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `.github/copilot-instructions.md`, `.cursor/rules`, README) as **memory proposals** → build index → generate **Repo Map** → snapshot Workbench. Show each real step with progress in S7. Ask the user to confirm detected commands when confidence is low.
3. Index pipeline (indexer service + indexing Workbench with read-only token): diff against last indexed SHA; tree-sitter parse changed files; syntax-aware chunking with path/symbol/line metadata; embeddings via gateway (record embedding model per chunk); store keyed `(repo_id, blob_sha, chunk_range)`; SCIP index (when build succeeds) to object storage; module summaries lazily regenerated when ≥ 20% of a module's files change.
4. Triggers: project creation, push webhook to default branch, mission start on unindexed base SHA, manual refresh.
5. Retrieval: hybrid (ripgrep on live tree or Postgres FTS) + vector + symbol-graph expansion → reciprocal-rank fusion → optional model rerank of top 50. Results always carry `file:line` and SHA; before editing, re-read from the live tree.
6. Limits: launch defaults 2 GB / 200k files; above limits, index user-selected paths and say so. Generated/vendored/binary files excluded from embeddings.
7. Repo Map content: structure overview, languages, frameworks, entry points, build/test/lint commands, key modules with summaries and citations, conventions from memory.

---

## 9. Missions (the code agent) — requirements

### 9.1 States

`CREATED → INTAKE → CLARIFYING? → CONTEXT → PLANNING → PLAN_REVIEW? → EXECUTING ⇄ CHECKPOINT_WAIT ⇄ STEERED → VERIFYING → REPORTING → COMPLETED | COMPLETED_UNVERIFIED`, plus `PAUSED`, `WAITING` (expired checkpoint or provider outage), `BLOCKED` (needs non-checkpoint user input such as a missing secret), `CANCELLED`, `FAILED`. Persist every transition to Postgres and emit `mission.state`.

### 9.2 Lifecycle behaviour

1. **Intake:** normalize objective, attachments, linked issue (issue/PR content is untrusted input); classify task type (bugfix, feature, refactor, tests, investigation, docs, dependency update) and complexity.
2. **Clarify:** if ambiguity is high, ask ≤ 3 questions in one structured checkpoint card. In Autonomous mode choose defaults and record assumptions in the Plan.
3. **Context:** fork the project's latest Workbench snapshot into a mission Workbench at the base ref; load Repo Map, pinned + retrieved memory (≤ 2k tokens), instruction files, retrieved code; run read-only **Scouts** for broad tasks (Phase 3).
4. **Plan:** steps `{id, title, intent, files_expected, commands_expected, evidence_expected, risk}`. Plan-first mode → PLAN_REVIEW checkpoint where the user can edit/reorder/delete/add constraints. Balanced mode shows the plan and auto-starts after 60 s unless opened.
5. **Execute:** tool loop. Each tool call is a durable step with idempotency key `missionId:stepSeq`. All calls go through `ToolRouter → Policy`.
6. **Steering:** user messages are injected at the next loop boundary with priority; the agent must emit a `status.summary` explaining how it adjusted.
7. **Verify (mandatory):** after the final edit, run detected tests (+ lint/type-check/build when detected); compare against base results (run on base if needed to classify pre-existing failures); self-review the full diff against objective and Plan; capture before/after screenshots for UI changes when runnable.
8. **Report:** generate the Mission Report artifact; every claim references a Trace event id; claims without tool evidence are marked Unverified.
9. **Handoff:** per autonomy, push the agent branch and open/update a **draft** PR whose body comes from the Report (respect the repo's PR template).

A mission may end `COMPLETED` only if: all Plan steps are done/skipped-with-reason/descoped by user; verification ran after the final edit with results attached; regressions vs base are zero or explicitly accepted by the user; self-review ran; UI evidence exists where applicable. Otherwise `COMPLETED_UNVERIFIED`, `FAILED` or `BLOCKED`.

### 9.3 Stop / ask / continue rules

- Verified achievement → stop and report.
- Ambiguity that changes a user-visible outcome → ask; continue independent steps meanwhile.
- Ambiguity with a reasonable default → continue; record assumption.
- Action above autonomy → checkpoint.
- Missing credential/secret/permission → `BLOCKED` with exact instructions.
- Budget at 80% → notify; at 100% → pause and ask to extend or wrap up with a partial report.
- Diff touching areas unrelated to the Plan → pause, explain, ask.

### 9.4 Loop and waste prevention (implement in `AgentLoop`)

- Budgets: max steps (default 150), max cost (user cap), max active wall-clock (default 2 h), max Workbench CPU-minutes.
- No-progress detector: fingerprint `(tool, normalized args, observation hash)`; 3 repeats → forced reflection step; 5 → stop and ask. Same failing test after 4 distinct fix attempts → stop and ask with diagnosis and options.
- Diff-churn detector: oscillating edits → reflection + checkpoint.
- Context compaction at 70% of the model window into a structured **Mission Journal** (facts, files touched, decisions, open questions); raw observations retrievable by id.
- Model escalation to the Reasoning role after repeated failures; de-escalation for mechanical steps.

### 9.5 Resumability

Snapshot the Workbench at checkpoints, before R2+ actions, and after 10 min paused. On resume, restore snapshot + state + Journal; the agent re-validates the tree (`git status`, quick test) before continuing. Workbench crash → retry the step once on a restored snapshot; Trace shows "Workbench restarted".

### 9.6 Agent tools (MVP set; ≤ 25 tools)

Each tool declares `name`, model-facing `description`, `input_schema` and `output_schema` (JSON Schema 2020-12), static `risk_class`, dynamic `risk_fn` (may raise, never lower), `timeout`, `max_output_bytes` (overflow → artifact reference), `redaction`, `idempotent`, `visibility`.

| Tool | Purpose | Static risk |
|---|---|---|
| `fs.list`, `fs.read` | List directories; read with line ranges | R0 |
| `fs.edit` | Exact string replace; fails with locations if not unique | R1 |
| `fs.write` | Create/overwrite file | R1 |
| `fs.patch` | Apply unified diff (no fuzz) | R1 |
| `search.text` | ripgrep | R0 |
| `search.symbols` | SCIP/tree-sitter symbols, definitions, references | R0 |
| `search.semantic` | Hybrid retrieval | R0 |
| `project.explain` | Area summary with citations | R0 |
| `shell.run` | Run command (cwd, timeout, env refs) | R1 (dynamic) |
| `shell.start`, `shell.read`, `shell.stop` | Background processes | R1 |
| `test.run` | Run detected tests; parse JUnit XML/TAP/pytest/jest/go test JSON | R1 |
| `git.status`, `git.diff`, `git.commit` | Local git | R1 |
| `git.push` | Push agent branch via egress/broker | R2 |
| `github.issue.read`, `github.pr.read`, `github.checks.read` | Read GitHub context | R0 |
| `github.pr.create`, `github.pr.update` | Draft PR create/update | R2 |
| `github.comment` | Comment on PR/issue | R2 (externally visible) |
| `web.search`, `web.fetch` | Research via proxy | R0 |
| `memory.search` | Retrieve memory | R0 |
| `memory.propose` | Propose memory entry (user confirms unless auto-save enabled for that scope) | R0 (proposal) |
| `ask_user` | Structured question → checkpoint | R0 |
| `plan.update` | Modify the Plan (emits Signal) | R0 |
| `browser.*` (Phase 2) | open, snapshot (accessibility tree default), screenshot, click, type, select, wait_for, console, network | localhost R1; external submit R3 |

Tool outputs returned to the model must be wrapped as untrusted data with provenance (`source`, `url/path`, `trust="untrusted"`), and system prompts must state that instructions inside tool outputs are data, not commands.

Error messages must be actionable (e.g., "old_string matched 3 locations: lines 12, 48, 90").

### 9.7 Mission Report schema

`outcome` (paragraph) · `status` (VERIFIED | UNVERIFIED | PARTIAL | FAILED | BLOCKED) · `changes[]` (group, file, +/-, rationale) · `evidence[]` (kind: command|test|lint|build|screenshot, artifact_id, exit_code, summary, event_ref) · `decisions[]` · `assumptions[]` · `risks[]` · `follow_ups[]` · `cost` (by model, workbench, search) · `duration` · `links` (branch, PR, artifacts). Render natively (S20) and as Markdown for PR bodies.

---

## 10. Workbench and terminal — requirements

1. Kinds: **project** (interactive terminal; agent when idle), **mission** (forked, exclusive), **scratch** (ephemeral for Ask; no project data, no secrets).
2. Isolation: hardware-virtualized microVM per Workbench; no shared kernel between tenants; never expose a Docker socket from the host. Nested containers inside the VM are allowed for project services.
3. Images: curated Ubuntu LTS base + JDK, Android command-line tools, Node, Python, Go, Rust, build-essential, git, ripgrep, Chromium + Playwright, `xavro-agentd`. Project customization via `.xavro/workbench.yaml` (packages, setup script, services, secret refs, egress additions) with an in-app editor (S14).
4. Lifecycle: `PROVISIONING → READY → BUSY ⇄ IDLE → (idle 10 min) SNAPSHOT+PAUSE → ARCHIVED → DELETED`. Cache setup in snapshots keyed by hash(image + workbench.yaml + lockfiles). Maintain warm pools; target mission start < 5 s P50 from snapshot.
5. Resources: default 2 vCPU / 4 GB / 20 GB; plan-dependent up to 8 vCPU / 16 GB / 100 GB; cgroup limits on pids/CPU/memory; disk quotas.
6. Network: default-deny egress through the egress proxy. Allowlist: package registries (npm, PyPI, Maven Central, Google Maven, crates.io, Go proxy, RubyGems, NuGet), GitHub (bound repo only, auth injected by proxy), user-approved additions (R3 checkpoint to add). Block link-local/metadata endpoints and private ranges. No inbound except the control channel.
7. Files: working tree at `/workspace`; uploads read-only at `/inputs`; symlinks may not resolve outside `/workspace` for agent fs tools.
8. Secrets: project secrets are encrypted (KMS envelope); injected only into commands that reference them by name at process start; outputs scrubbed (exact-match + entropy patterns) before reaching the model, logs, or Signals. **No long-lived Xavro, GitHub or provider credential ever enters a Workbench.**
9. Process management in `xavro-agentd`: process groups, PTY, sequenced stdout/stderr streaming, exit codes, resource usage, background handles; cancellation SIGINT → 5 s → SIGTERM → 5 s → SIGKILL; per-command timeout default 10 min, max 60 min.
10. Git hooks disabled for agent git operations by default (`core.hooksPath` → empty dir) unless the project enables them.
11. User terminal commands are user actions (no agent approvals) but are subject to egress policy and recorded in the audit log.
12. Abuse detection: sustained full CPU with unknown egress attempts, known miner signatures → suspend and alert.
13. Failure: provider outage → missions `WAITING` with banner, auto-resume; setup failure → `BLOCKED` with the failing line and logs.

---

## 11. GitHub — requirements

1. Integrate only via the **Xavro GitHub App**. Requested permissions: Metadata (R), Contents (R/W), Pull requests (R/W), Issues (R/W), Checks (R), Actions (R), Commit statuses (R). Do **not** request Workflows at launch.
2. Use GitHub App user-to-server tokens to list accessible installations/repos and to **verify the requesting user has push access before every write on their behalf**. Never act on a repository where the user lacks push access.
3. Installation tokens are minted by the Credential Broker per operation, down-scoped to the single repository and minimum permissions, and discarded after use. They never reach the Workbench disk/env/git config or any model context.
4. Branches: `xavro/<slug>-<shortid>`. Never push to default or protected branches; never force-push to branches the agent did not create; **never merge** — merge is R4, user-only, via a dedicated confirmation showing checks and target branch (or in GitHub).
5. Commits: small, one per plan step where natural, App-authored (API-signed, verified), `Co-authored-by` the requesting user (configurable).
6. PRs: draft by default; body from Mission Report; link back to the mission; respect PR template.
7. Issues: list/detail; "Start mission from issue" pre-fills objective (marked untrusted).
8. Review comments and check failures flow into the mission as events; user may let Xavro address them (follow-up iteration on the same branch) or "drive PR to green" when autonomy allows.
9. Webhooks: installation, installation_repositories, push, pull_request, pull_request_review, pull_request_review_comment, issue_comment, issues, check_run, check_suite, workflow_run. Verify `X-Hub-Signature-256`; dedupe by delivery id; process via outbox/queue; redeliver on gaps.
10. Cache PR/issue metadata with ETags; GitHub is the source of truth.
11. Cross-visibility rule: content from private sources may not be written to public repositories without an R3 checkpoint.
12. Idempotency: before creating a PR, look up an existing PR by head branch; every GitHub write carries an idempotency key.

---

## 12. Browser — requirements (Phase 2)

Playwright + Chromium in the mission Workbench; fresh profile per mission; downloads confined to `/workspace/.downloads`; observations default to accessibility snapshots with element refs, screenshots on demand; localhost actions R1; external navigation read R1; external form submission R3 with screenshot and destination origin in the checkpoint; entering credentials is forbidden for the agent. Live screenshot stream (1–2 fps) in the Mission Browser pane. Open-web interactive browsing remains EXPERIMENTAL behind a flag.

---

## 13. Memory — requirements

1. Scopes: personal (user), project, conversation (history + compaction summaries), mission state (Plan, Journal, observations), execution state (Workbench).
2. Every entry shows content, source (conversation/mission/file), timestamps, pinned flag; users can edit, delete, pin.
3. Agent writes are **proposals** shown in the Mission Report and S13; auto-save is opt-in per scope.
4. Entries ≤ 500 characters; pinned entries always injected; others retrieved by hybrid relevance within a ≤ 2k-token budget.
5. Conflicting proposals display as updates (diff), not duplicates. Entries referencing missing files/symbols are flagged stale.
6. Deletion removes from retrieval immediately and from storage (including embeddings) within 24 h; account deletion within 30 days.
7. Memory is never used for training and never shared across users.
8. If memory retrieval fails, proceed and emit a `notice` Signal ("Memory unavailable"); never invent remembered facts.

---

## 14. Research mode — requirements (Phase 2)

1. Scope card: restated question, sub-questions, source types, depth (Quick / Deep) with cost estimate, output format. User edits or accepts.
2. Runs as a mission of type `research` (no Workbench write tools) so it survives app closure and notifies on completion.
3. Research Trace: queries, sources found, read, rejected (with reason).
4. Ranking by relevance, authority (primary > official docs > reputable publications > forums), freshness, diversity.
5. Claim table `(claim, source, quote, date)`; conflicting claims surfaced.
6. Citation verifier: every factual sentence maps to ≥ 1 claim; quoted passages must exist in fetched text; unsupported sentences rewritten as uncertain or removed.
7. Output: report artifact with executive summary, findings per sub-question, comparison tables, confidence per claim, bibliography with access dates, open questions. Saved to Library and optionally to a project; follow-ups reuse the source set.

---

## 15. Model gateway — requirements

1. Registry (database-backed, admin-editable): provider, model id, display name, capabilities (text, vision, audio, tools, parallel tools, structured output, reasoning controls, prompt caching, max context/output, grounding), pricing (input/output/cache), measured latency (rolling P50/P90 TTFT and throughput), Xavro eval scores per task type, data policy (retention, training, regions), status, allowed plans.
2. Roles: Fast, Chat, Reasoning, Coding, Research, Vision, Embedding. Each role has an ordered candidate list.
3. Routing: filter by required capabilities, user/plan restrictions, data-policy constraints, provider health, budget; score by quality × cost × latency weights from the user's preference (Faster/Balanced/Best).
4. Fallback: per-provider circuit breakers; failover to an equivalent-capability candidate; mid-stream failure in Ask continues on the fallback with partial output as context; missions retry the durable step.
5. Normalized internal message/tool/stream format; adapters convert. Validate every tool call against its schema before execution; invalid calls are returned to the model as errors.
6. Provider reasoning tokens are never displayed; opaque/encrypted reasoning blocks are passed back to the same provider only where required for continuity.
7. Metering: every call writes the usage ledger (tokens, cache hits, cost in micros, model, role, user, project, mission).
8. Prompt caching keys never mix tenants.
9. BYOK (Phase 3): keys encrypted via KMS envelope encryption, used only inside the gateway via the broker, usage shown separately.
10. On-device Gemini Nano (ML Kit GenAI): experimental flag only, for private on-device conveniences (e.g., notification summaries), never for answers or agent work.

---

## 16. Files, artifacts and Library — requirements

Artifact record: `id, kind (code_file|diff|document|report|image|log|dataset|archive), title, owner, project?, conversation?, mission?, version, mime, size, storage_key, preview, provenance, retention_class`. Link artifacts to the producing message/step. Library (S23): filters, full-text search, native viewers per kind (PdfRenderer for PDF, virtualized monospace for logs, table preview for datasets). Version text artifacts with diffs between versions. Scan uploads for malware before parsing; parse in network-less sandboxed workers; enforce per-plan size limits. Sharing via signed expiring links (Phase 4), never public by default.

---

## 17. Signals (real-time) — requirements

1. Schema in `xavro-protocol`: `{stream, seq, ts, type, actor, visibility, data, refs}`; streams `conversation:<id>`, `mission:<id>`, `workbench:<id>`, `user:<id>` (inbox).
2. Types: `message.delta`, `message.completed`, `status.summary`, `plan.updated`, `step.started|completed|failed`, `tool.call.started|progress|completed|failed`, `terminal.output`, `fs.changed`, `test.result`, `browser.frame`, `checkpoint.requested|resolved|expired`, `git.commit`, `github.pr.opened|updated`, `budget.updated`, `mission.state`, `error`, `notice`.
3. Persist every user-visible Signal to `run_events (stream, seq)` before publishing to Valkey pub/sub.
4. Transport: SSE over HTTP/2 with `Last-Event-ID` replay; WebSocket only for terminal PTY; FCM with ids only (app fetches content via authenticated API).
5. Coalesce `message.delta` and `terminal.output` to ≤ 20 Hz for mobile; full fidelity remains in artifacts.
6. `status.summary` lines are generated from tool calls and plan state, never from hidden reasoning.

---

## 18. Permissions and policy — requirements

### 18.1 Risk classes

- **R0 Observe:** reads within the user's scope; automatic.
- **R1 Local change:** changes confined to the mission Workbench; automatic (Careful: checkpoint first shell command per mission).
- **R2 Shared but reversible:** push agent branch, open/update draft PR, comment, install new dependency from an allowlisted registry, enable read-only MCP tool.
- **R3 Consequential:** mark PR ready, request reviewers, add egress domain, external form submit, modify CI workflows, use secrets in new commands, delete agent-created branches.
- **R4 Destructive/irreversible/privileged:** merge, force-push, delete content outside agent branches, deploy to production, change repository settings, use credentials on websites. **User-only**; the agent may only request.

Dynamic risk patterns raise class (e.g., `rm -rf` outside workspace → R3; `curl … | sh` → R3; non-allowlisted registry → R3; `git push --force` → R4).

### 18.2 Autonomy levels

| Level | Plan | R1 | R2 | R3 | R4 |
|---|---|---|---|---|---|
| Careful (default for first 3 missions) | Approval required | Checkpoint first shell use | Checkpoint | Checkpoint | User-only |
| Balanced (default after) | Shown; auto-start after 60 s | Auto | Push/draft PR auto + notify; comments checkpoint | Checkpoint | User-only |
| Autonomous | No approval | Auto | Auto | Checkpoint unless pre-approved type for the project | User-only |

### 18.3 Enforcement

1. `Policy.decide(principal, action, resource, context) → Allow | Checkpoint(reason) | Deny(reason)` runs inside `ToolRouter` for every tool call and inside API handlers for every user action.
2. Checkpoint decisions are stored with user, device, time, payload hash and a server signature; execution verifies the hash of the payload about to run equals the approved hash.
3. Grants: "Approve similar for this mission" stores a pattern scoped to the mission; project-level grants only via Project Settings.
4. Checkpoint batching within a short window; expiry default 24 h → `WAITING`; never auto-approve on expiry.
5. R2 actions expose "Revert" in the Trace where possible (close PR, delete agent branch, delete comment).
6. Push notification actions (Approve/Decline) are allowed only for R2; R3 requires opening the app; R4 requires step-up authentication (passkey/biometric).
7. Policy rules have table-driven tests covering every (action × risk × autonomy) combination.

---

## 19. Security — requirements

Design every boundary assuming the model has been fully hijacked by content it read. A hijacked agent must not be able to exceed its mission's granted permissions inside its Workbench.

1. **Identity:** passkeys first; Sign in with Google; email magic link fallback; no passwords. Access tokens ≤ 15 min; refresh tokens rotate with reuse detection and are bound to a device key in Android Keystore (proof-of-possession). Step-up authentication for R4 actions and security settings. Sign-in anomaly detection.
2. **Authorization:** every data access goes through a repository layer that requires a principal and sets a Postgres session variable used by Row-Level Security policies. Generate IDOR tests per endpoint.
3. **Prompt-injection containment:** untrusted-data wrapping with provenance; instruction hierarchy in system prompts; permissions independent of model intent; default-deny egress; ground-truth checkpoint payloads; an injection classifier that flags (not blocks) suspicious tool outputs and surfaces a Trace warning.
4. **Credentials:** Credential Broker isolated (separate service account, subnet, minimal IAM); only it can read the GitHub App private key and provider/BYOK keys; all mints and uses audited.
5. **Secrets hygiene:** secret scanning on clone with warnings; output redaction before model/log/Signal; KMS envelope encryption at rest; never log secrets or tokens.
6. **Repository attacks:** all execution in microVMs without credentials; git hooks disabled by default; symlink confinement.
7. **Documents and web:** parsing in network-less sandboxed workers with size/page limits and malware scanning; fetch proxy with SSRF protection (block private ranges, link-local, metadata, DNS rebinding).
8. **MCP (Phase 3):** admin-curated allowlist; pinned versions; tool description hashes — any change requires re-approval; calls through the MCP proxy with egress control and logging; remote servers via OAuth with minimal scopes.
9. **Supply chain:** lockfiles and pinned versions; Renovate with human review; CycloneDX SBOMs for images and APK; cosign signatures and provenance attestations (target SLSA L3); minimal base images; Play App Signing; CodeQL, OSV-Scanner, gitleaks in CI.
10. **Tenant isolation:** tenant-scoped cache keys, prompt caches, vector namespaces, encryption contexts.
11. **Audit:** append-only `audit_events` for authentication events, every R2+ action, every checkpoint decision, policy denials, admin actions; daily export to object-locked storage; user-visible Agent activity log in Settings → Security.
12. **Kill switches:** per mission (user), per project (user/admin), per tool, per provider, global (ops).
13. **Mobile:** Keystore-backed encrypted token storage; certificate transparency; no user-added CAs in release; optional `FLAG_SECURE`; editor WebView loads only bundled assets; verified App Links; PKCE + state for OAuth.
14. **Process:** threat model per feature in `docs/threat-model/`; external penetration test before public launch; bug bounty after launch.
15. **Abuse:** Play Integrity signals for risk scoring, new-account resource limits, GitHub write rate limits, anomaly detection on cost and egress.

---

## 20. Developer API and MCP server (Phase 5)

Missions API, Projects API, Ask API, Model Gateway API; an MCP server exposing projects and missions as tools/resources. OAuth 2.1 for third-party apps; API keys (`xv_live_…`, hashed, shown once, scoped, expirable, rotatable) for server-to-server. Scopes such as `missions:write`, `projects:read`, `checkpoints:answer:r2`; API keys can never answer R3 unless an org admin policy allows; R4 never. SSE with the same Signal schema; signed webhooks with retries. Token-bucket rate limits per key and org; cost headers. Date-based versioning (`Xavro-Version`). OpenAPI 3.1 generated from route definitions; SDKs for TypeScript, Python, Kotlin. Do not build this before Phase 5, but keep internal APIs clean enough to expose.

---

## 21. Web Console — requirements

TypeScript web app using the same APIs. Phase 1: internal operations only (provider status, model registry editing, feature flags, workbench pool health, abuse queue, user lookup with privacy controls). Phase 4: organization admin (members, roles Owner/Admin/Member/Viewer/Billing, SSO, policies, MCP allowlist, pre-approved R3 types, budgets and alerts, usage breakdown, audit search/export, kill switches, repository bindings). Internal admin requires WebAuthn hardware keys, just-in-time elevation, and full audit. Support staff cannot read user content without a time-boxed, user-approved access grant.

---

## 22. Observability — requirements

1. OpenTelemetry tracing from app request through API, gateway, provider call, tool, and Workbench command (W3C trace context; MCP tracing keys). One trace per user request and per mission step.
2. GenAI semantic conventions for model/agent/tool spans via `telemetry-genai` (pinned convention version).
3. Prompts/completions are not stored in general telemetry; Langfuse stores them only for internal eval traffic, opted-in users, or under a support grant, retention ≤ 30 days.
4. Metrics to implement: app start/jank/ANR/crash-free; TTFT, tokens/s, latency by model/provider; tool-call validity, refusal, fallback, truncation rates; steps per mission, verified rate by task type, checkpoint count and rejection rate, no-progress stops; per-tool latency/error/timeout; Workbench start times, resource usage, OOMs; cost per message/mission/verified outcome; North Star.
5. SLOs: API availability 99.9%; SSE delivery; mission step success. Multi-window burn-rate alerts; provider degradation; pool exhaustion; cost and egress anomalies; policy-denial and injection-detection spikes.
6. Product analytics events are allowlisted and contain no user content.

---

## 23. Evaluation — requirements

Build `evals/` with Inspect from Phase 0.

1. **Xavro-Missions:** grow to 300+ tasks across 30+ pinned repositories (Kotlin/Android, TypeScript/React, Python, Go, Java, Rust) with hidden verification tests and rubrics (50 in Phase 0, 150 in Phase 1).
2. Terminal tasks (setup, build fixing, dependency upgrades).
3. Repository understanding Q&A scored by file:line match.
4. Tool selection single-step tasks.
5. Safety & injection suite: 200+ adversarial fixtures (malicious README, issue comments, web pages, MCP tool descriptions, documents). Track attempted-violation rate and blocked-by-policy rate separately.
6. Research: citation precision/recall, accuracy, freshness.
7. Memory precision and harm.
8. Long-running reliability with forced pause, Workbench crash, provider failover; verify no duplicated side effects.
9. Metrics: verified pass@1, cost, steps, wall-clock, checkpoint count, unnecessary-checkpoint rate, unsupported-claim rate in Reports, recovery rate.
10. LLM-as-judge only with rubrics validated against human labels (≥ 0.8 agreement); never sole gate for safety.
11. **Release gate:** any change to prompts, tool schemas/descriptions, routing, or models must not regress verified pass@1 by > 2 points or any safety metric, without explicit human sign-off. Smoke suite on PRs touching agent code; full suite nightly.

---

## 24. Data model — implement

PostgreSQL (OLTP) with UUIDv7 ids, RLS on all tenant-owned tables, forward-only expand/contract migrations, PITR backups, quarterly restore drills.

```
organizations, users, devices, identities_linked, github_installations,
projects, project_repositories, workbench_definitions, workbenches, project_secrets,
conversations, messages (parent_id, branch_index), missions, plan_steps,
tool_calls (args redacted + hash), checkpoints (payload_hash, signature, expires_at),
grants, run_events (PK stream+seq, monthly partitions), memory_entries (state: active|proposed|stale|deleted),
artifacts, usage_ledger (cost_micros), api_keys (hash), audit_events (append-only),
notifications, model_registry
```

Column-level definitions follow the blueprint §26.2; add indexes for every foreign key and every list query used by the app. Vector database (separate Postgres + pgvector, HNSW): `chunks(repo_id, blob_sha, range, path, symbol, lang, text, embedding, embed_model)`, `memory_vectors(entry_id, embedding)`, partitioned/namespaced by tenant. Object storage buckets: attachments, artifacts, logs, indexes, snapshots (if self-hosted), audit-export (object lock). Retention: `run_events` 90 days hot then summarized; Mission Reports until user deletion; Langfuse content ≤ 30 days.

---

## 25. Infrastructure and deployment — implement

1. OpenTofu modules for: VPC, Cloud SQL/AlloyDB (HA), vector Postgres, Valkey, buckets (with lifecycle and object lock), Secret Manager, KMS keys, Cloud Run/GKE Autopilot services, Restate cluster (3 nodes, persistent disks) or Restate Cloud, Envoy egress proxy, Cloud Armor, OTel Collector, Grafana stack (or Grafana Cloud), Langfuse, Sentry/PostHog configuration.
2. Services: `api`, `gateway`, `agent-worker`, `restate`, `broker`, `egress`, `indexer`, `parser` (gVisor), `notifier`, `console`; Workbench provider via `WorkbenchProvider`.
3. Network: public HTTPS LB + WAF for `api` and `console`; everything else private; broker reachable only from `api` and `agent-worker`; Workbench egress only via the egress proxy.
4. CI (GitHub Actions): lint (ktlint/detekt, eslint), unit tests, Android UI + screenshot tests on Gradle Managed Devices (phone, foldable, tablet), server integration tests with Testcontainers (Postgres, Valkey, Restate), protocol compatibility tests, eval smoke on agent changes, security scans, SBOM + signing. Deploy with OIDC workload identity — no static cloud keys.
5. Environments: dev, staging (prod-like with synthetic users and a dedicated GitHub test organization), production. Canary 5% → 25% → 100% with automatic rollback on SLO burn.
6. Android release: internal track on every main merge; weekly closed testing; staged production rollout 1% → 10% → 50% → 100% gated on crash-free sessions; in-app updates for critical fixes; Play data safety and AI-content policies satisfied.
7. Feature flags via OpenFeature with a self-hosted provider; app remote config payloads signed.
8. Billing: Google Play Billing for consumer subscriptions on Android; web billing for teams (Phase 4); usage ledger drives limits and invoices.

---

## 26. Testing — implement

- Unit (JUnit 5/Kotest, coroutines-test, Turbine) for domain, policy, routing, parsers, reducers.
- Policy matrix tests: 100% rule coverage, property tests for dynamic risk patterns.
- Server integration tests with real Postgres/Valkey/Restate via Testcontainers; external providers faked only at the network boundary with recorded contracts, each matched by a live nightly test. **Do not mock systems Xavro owns in integration tests.**
- Protocol compatibility tests (old app ↔ new server and vice versa within the support window).
- Android: Compose UI tests and screenshot tests across width classes, dark/light, 200% font, RTL; accessibility checks; Macrobenchmark performance gates.
- E2E: sign-in → project → GitHub test org repo → mission → draft PR, on staging (Maestro or UI Automator).
- Security: SAST, dependency, secret scanning, DAST, authorization tests, sandbox isolation tests (cross-tenant FS/network, egress, resource limits, teardown), injection suite, pre-launch pentest.
- GitHub integration: webhook signature, duplicate delivery, token scoping, push-access verification, PR idempotency.
- Load: SSE fan-out, concurrent missions, gateway throughput before each phase launch (k6/Gatling).
- Chaos: provider outage, Workbench crash, Restate node loss, DB failover; verify idempotent side effects.

---

## 27. Honest-state catalogue (implement these exact behaviours)

| Situation | Required UI |
|---|---|
| No model provider configured in an environment | Ask shows "No AI model is configured for this environment" with admin instructions; composer disabled. |
| GitHub App not installed / repo not covered | Project shows "Xavro doesn't have access to this repository" with Install/Configure button. |
| User lacks push access | Mission start blocked: "You need push access to <repo> to run missions on it." |
| Workbench provider unavailable | Missions/terminal show "Workbench is temporarily unavailable" with status and auto-retry; no simulated output. |
| Index not ready | Code-aware features labelled "Indexing (n of m files)"; fallback search labelled "Text search only". |
| SCIP unavailable | "Precise navigation unavailable for this project" + reason (build failed / unsupported language). |
| Search provider down | Research shows partial results with "Some sources couldn't be searched". |
| Push permission denied | Notification-dependent features explain that Xavro can't notify; Inbox still works. |
| Feature not yet released | Not shown at all. Never show disabled placeholder screens for unbuilt features. |

---

## 28. Definition of Done (applies to every feature)

1. Loading, empty, error (specific cause), retry, offline, cancellation (where applicable), permission-denied and unavailable states implemented.
2. No fake data, fake progress, fake success or placeholders in production builds.
3. Server-side authorization and policy enforced; policy tests added; threat-model delta written.
4. Accessibility: TalkBack labels, focus order, contrast, 200% font, 48 dp targets, reduced motion.
5. Adaptive: compact/medium/expanded verified, foldable postures, landscape, multi-window, RTL.
6. Telemetry: traces, metrics and allowlisted product events; dashboards updated.
7. Tests: unit + integration + UI/screenshot; agent-affecting changes pass the eval gate.
8. Performance within §6 budgets and service SLOs.
9. Recovery: process death, network loss, server restart, duplicate delivery handled idempotently.
10. Docs: user help text, ADRs, runbooks updated.

---

## 29. Build order

### Phase 0 — Foundation (start here)

Build in this order, each step with tests and CI before moving on:

1. Monorepo skeleton (§3), `AGENTS.md`, ADRs for every decision in §2, CI pipelines, OpenTofu dev environment.
2. `xavro-protocol` with Signal schema, core DTOs, error codes; compatibility test harness.
3. Meridian design system (tokens, typography, core components, Trace Rail) with screenshot tests in all width classes and both themes.
4. Identity: Zitadel integration, session exchange, device registration, Credential Manager sign-in on Android (S1, S2).
5. API skeleton: persistence layer with RLS, `Policy` module with the full risk/autonomy matrix and tests, audit log.
6. ModelGateway with Anthropic, OpenAI and Google adapters, registry, routing, fallback, metering; Ask (S3, S4) end to end with SSE resume.
7. `MissionEngine` on Restate (and a Temporal spike behind the same port; record the comparison in an ADR).
8. `WorkbenchProvider` integration after a vendor spike comparing ≥ 2 managed microVM providers (isolation, start latency, snapshot/resume, egress control, PTY streaming, regions, compliance, cost); `xavro-agentd`; base image; egress proxy; Credential Broker with GitHub App.
9. Observability baseline and `evals/` with the first 50 tasks and 50 injection fixtures.

**Exit:** internal user signs in on Android, chats with streaming and fallback; an internal mission edits a test repo in a Workbench, runs tests and pushes a branch through the broker, with a complete trace.

### Phase 1 — MVP ("Fix it from your phone")

Projects (S5–S9, S11 read, S13–S15 basic), GitHub App connection and repository binding, Workbench preparation + Repo Map (minimal index: tree-sitter outline + ripgrep), Missions end to end (S16–S21) with Plan, Trace, Trace Rail, Checkpoints, Changes, Terminal (read-only stream), Mission Report, draft PR; budgets and cost meter; Inbox (S24) and FCM; Settings essentials (S26); adaptive layouts; honest states (§27); security review + external pentest.

**Exit:** ≥ 45% verified completion on Xavro-Missions (150 tasks); ≥ 60% beta acceptance of completed missions; mission start < 10 s P50; crash-free sessions ≥ 99.5%; no open critical security findings.

### Phase 2 — Public launch

Full indexing (embeddings, hybrid retrieval), interactive terminal (S27, S14), Research mode (S22), steering, PR review iterations, browser verification, Library (S23), global search (S25), offline cache, Arabic + RTL, Play Store launch.

### Phase 3 — Advanced

MCP client with allowlist, SCIP navigation, edit mode, Scouts, BYOK, project-level grants, drive-PR-to-green, multi-repo projects, three-pane mission and multi-window missions.

### Phase 4 — Professional

Teams, shared projects, Web Console org admin, roles, SSO, audit export, Cedar policies, artifact sharing, self-hosted Firecracker pool, EU region.

### Phase 5 — Flagship platform

Developer API, Xavro MCP server, webhooks, SDKs, enterprise (SCIM, residency, BYO-cloud Workbenches, private model endpoints), scheduled missions, deployment tools under R3/R4, A2A if adoption justifies, additional clients via shared KMP modules.

---

## 30. Explicit prohibitions

- Do not run agent work, builds, tests, or user code on the Android device.
- Do not use plain containers as the isolation boundary for Workbenches.
- Do not place any long-lived credential in a Workbench, in model context, in logs, or in the Android app (except the user's own Xavro refresh token in Keystore-backed storage).
- Do not let the agent merge, force-push, deploy to production, change repository settings, or enter credentials on websites.
- Do not rely on prompts to enforce permissions.
- Do not display or store hidden model reasoning.
- Do not add engagement mechanics (streaks, gamification, unsolicited notifications).
- Do not ship disabled placeholder screens for unbuilt features.
- Do not adopt a third-party agent framework into the runtime, or a gateway product that hides provider features, without an ADR that justifies replacing the corresponding Xavro-owned component.
- Do not merge pull requests into `main` without human review.

**Begin with Phase 0, step 1.**
