# Personal Brain — Architecture

> Technical design for the Personal Brain platform. This document describes **how**
> the platform is structured to satisfy the product requirements in
> [`SPEC.md`](SPEC.md). It is the companion to that document: `SPEC.md` says *what*
> the product does and *why*; this document says *how* it is built. See
> [`CLAUDE.md`](../CLAUDE.md) for how we work on these documents.

## 1. Purpose and relationship to the spec

`SPEC.md` is a requirements document, deliberately written at requirements altitude:
it describes behavior, capabilities, and constraints, and avoids implementation
detail. This document picks up where it leaves off and describes the **architecture** —
the components the platform is decomposed into, what each is responsible for, the state
each owns, how they interface, and how requests and data flow between them.

Two ground rules shape everything here:

- **Tech-agnostic.** This document names **capabilities**, not products. It says "a
  semantic index" or "an object store," never a specific database, framework, language,
  model provider, or PaaS vendor. Those selections are deferred (see
  [Open technical questions](#10-open-technical-questions)); the architecture is
  designed so that they can be made later without reshaping the system.
- **Traceable.** Every architecturally significant decision cites the `FR-`/`NFR-`
  handle from `SPEC.md` that drives it. This document does **not** restate
  requirements; it references them and explains how the structure meets them. When the
  two documents disagree, `SPEC.md` is the source of truth and this document is
  corrected.

This is a **whole-system overview** at a single, consistent altitude. Deeper treatment
of individual subsystems (their internal data models, algorithms, and contracts) is
left to later, focused design documents.

### Vocabulary

This document reuses the product vocabulary defined in `SPEC.md` exactly — *knowledge
base, node, section, workspace, project, task, orchestrator, agent, tool, connector,
source artifact, review queue, project↔task mapping*. It introduces a small number of
**architecture-level** terms for components the spec implies but does not name; each is
defined once, where it first appears, and used consistently thereafter:

- **Knowledge store service** — the component that owns and hides the underlying
  store of the knowledge base.
- **Effect tracker** — the component that records the per-effect status of an input's
  outcomes; it backs the durable review queue.
- **Model gateway** — the component through which all agent model calls pass, so model
  sourcing is selectable per agent.
- **Connector adapter** — the platform-side component that drives a task connector and
  owns the project↔task mapping and reconciliation.

## 2. Architectural drivers

The architecture is shaped less by features than by a handful of cross-cutting forces.
Each is a requirement that constrains structure, not just behavior.

- **The knowledge base is the single source of truth (FR-6, NFR-2).** All project and
  workspace knowledge lives in one authoritative store; no component keeps private
  knowledge beside it. This is why agents can be made stateless and why everything an
  agent "knows" is, by construction, visible to the user.

- **Storage mediation is always present; agent mediation is optional (FR-42, FR-45).**
  Neither the user nor the agents ever touch the underlying store directly — a single
  service mediates it. But the agent is *optional* in the loop: hand-editing and
  agent-driven change are two paths to the **same** mediated store.

- **Agents hold no private state, so services can be stateless (FR-6, NFR-6).** With
  all authoritative state in managed storage, the compute tier (application service,
  orchestrator, agents) can be horizontally scalable and disposable. This drives a
  clean split between **stateless compute** and **authoritative managed state**.

- **Model-agnostic, per-agent model sourcing (FR-20, NFR-4, NFR-7, NFR-10).** Different
  agents may use different models, hosted by third parties or self-hosted. This forces
  model access behind a single indirection (the model gateway) rather than letting
  agents call providers directly.

- **Two systems, eventual consistency (FR-52–FR-56).** A single input can change both
  the knowledge base (a store the platform controls) and the external task platform (one
  it does not). Cross-system atomicity is impossible, so the architecture is built
  around **best-effort, per-effect application with tracked status** and a convergence
  mechanism, not transactions.

- **Responsiveness and never losing work (NFR-14, FR-50, FR-53).** Interactive inputs
  must feel conversational; heavy inputs must not block and must never be lost. This
  splits capture into a **synchronous fast path** and an **asynchronous durable path**,
  and makes the review queue durable.

- **Single-user scale with first-class semantic retrieval (NFR-13, FR-61).** Hundreds
  of projects, thousands of sections; a single input routinely spans several projects.
  Retrieval assembles a working set of *sections* across projects rather than loading
  whole projects, which makes a **semantic index** a core component, not an add-on.

- **Collaboration-ready, but single-user now (NFR-5, NFR-15).** The current product is
  single-user with account-global configuration, but no structural choice may preclude
  later multi-user sharing at project or workspace granularity. This keeps identity and
  tenancy boundaries explicit even though they are trivial today.

## 3. System context and boundary

At the outermost level, the platform sits between the user and two classes of external
system.

```
        ┌─────────────────────────── Personal Brain (platform boundary) ───────────────────────────┐
        │                                                                                            │
 User ──┼──▶ Thin clients ──▶ Application service ──▶ Orchestrator ──▶ Agent runtime                 │
 (typed │        (web,            (stateless)          (control        │                             │
  voice,│        mobile)                                plane)         ▼                             │
  docs, │                                                          Tool layer (MCP) ── autonomy gate │
  links)│                                                              │            │                │
        │                          Knowledge store ◀──────────────────┘            │                │
        │                          Retrieval/index                                 │                │
        │                          Capture pipeline                                │                │
        │                          Review queue / effect tracker                   │                │
        │                          Connector adapter ───────────────┐  Model gateway│                │
        │                          Source-artifact store            │              │                │
        └────────────────────────────────────────────────────────┼──────────────┼────────────────┘
                                                                   ▼              ▼
                                                      External task platform   Model providers
                                                       (via connector / MCP)   (3rd-party APIs &
                                                                                self-hosted models)
```

**Inside the boundary:** the thin clients' presentation logic, the stateless compute
tier (application service, orchestrator, agents, tool layer), and all authoritative
state (knowledge base, project↔task mapping, review queue, settings) plus
interaction/source state (conversation log, source artifacts).

**Outside the boundary:** the **external task-management platform**, reached only
through a configurable **connector** (FR-16, MCP the intended mechanism); and **model
providers**, whether third-party hosted APIs or self-hosted open models — the latter
run on the same managed infrastructure but are still treated as a provider behind the
model gateway (NFR-7).

**Actors:** a single user (today), interacting through thin web and mobile clients
(NFR-6) by typing, speaking, sharing documents, or providing links (FR-8). Voice is
input only (FR-36).

## 4. Logical architecture — components and responsibilities

Each component below lists its **job**, the **state it owns** (if any), and its
**interfaces** to neighbors. The recurring split is between **stateless compute** (owns
no authoritative state) and the few components that **own managed state**.

### Client surfaces
*Job.* Present the product through thin web and mobile clients (NFR-6): a single
conversation surface for agent-directed input (FR-26), a **structured outline editor**
for direct hand-editing of the node tree (FR-45), a review-queue surface for confirming
or retrying proposed/failed effects (FR-47), and settings (autonomy, connector,
model/privacy — FR-21, NFR-15). Voice capture records audio for server-side
transcription (FR-48); clients do no transcription themselves.
*State.* None authoritative. Clients are thin; all logic and state live server-side.
*Interfaces.* Talk only to the application service.

### Application service
*Job.* The stateless backend entry point. Terminates client sessions, authenticates
requests (NFR-8), and routes them: agent-directed input goes to the orchestrator;
direct hand-edits go to the knowledge store service; review-queue actions go to the
effect tracker. It streams responses back to clients (conversational first response,
NFR-14).
*State.* None authoritative (sessions are short-lived; durable state lives downstream).
*Interfaces.* Clients (inbound); orchestrator, knowledge store service, effect tracker,
capture pipeline (outbound).

### Orchestrator
*Job.* The single control plane through which the platform orchestrates its agents
(FR-19) for everything the user directs to them (FR-26). For each input it
(1) **resolves the target project(s)** from explicit indication or inference, asking the
user when confidence is insufficient (FR-27); (2) **selects and invokes** the agents the
input requires, interpreting the input so the user need not structure it (FR-9, FR-28);
(3) **combines** their outputs into a set of intended effects — a single input may yield
several (FR-11) — spanning knowledge-base writes and task changes; and (4) **applies**
them through the tool layer under the configured autonomy (FR-28, FR-29). Extraction
and writing are capabilities it coordinates, not separate agents (FR-33). Creating or
reorganizing workspaces and projects is likewise agent-directed input through the
orchestrator (FR-38), or done directly by hand (FR-41). It owns
the in-flight state of an exchange (a pending clarification, an assembled proposal) but
persists anything that must outlive the exchange to the review queue.
*State.* In-flight exchange state only; durable interaction state (pending proposals,
clarifications) is delegated to the effect tracker/review queue (NFR-11).
*Interfaces.* Application service (inbound); agent runtime, tool layer, effect tracker
(outbound). No agent writes the knowledge base except through the orchestrator and tool
layer (FR-29).

### Agent runtime
*Job.* Execute the specialized agents the orchestrator selects. The committed core is
the **retrieval agent** (finds relevant projects/sections; the orchestrator relies on
it to resolve the target project — FR-32, FR-34) and the **Q&A agent** (assembles
answers from the knowledge base — FR-32). The roster is extensible with further agents
by domain or kind of work (FR-35), specialized along two axes that may combine (FR-24)
and never tied to an individual project (FR-25). Agents **read** through the tool layer
and **return results** to the orchestrator; they never write the knowledge base
directly (FR-29).
*State.* None. Agents keep no private knowledge (FR-6); their only durable grounding is
the knowledge base, with recent conversation as immediate context only (NFR-11).
*Interfaces.* Orchestrator (inbound/outbound); tool layer (for reads); model gateway
(for inference).

### Tool layer (MCP)
*Job.* Expose every read/write operation over the knowledge base and the task platform
as a **tool** (FR-23, MCP the mechanism). It is the single choke point where the
**autonomy gate** lives: each tool invocation is checked against the per-tool autonomy
setting (FR-21) and either executes immediately or is held as a proposal for
confirmation (FR-22). Autonomy is per-tool and uniform across agents — it is enforced
here, not inside any agent (FR-21).
*State.* None authoritative; it reads the autonomy configuration (account-global,
NFR-15) and enacts decisions against downstream stores.
*Interfaces.* Orchestrator and agents (inbound); knowledge store service and connector
adapter (outbound); effect tracker (to enqueue proposals and record effect status).

### Knowledge store service
*Job.* **Own and hide** the underlying store of the knowledge base (FR-42). It is the
*only* component that touches that store — for both agent-mediated writes and direct
hand-edits (FR-45). It models the **node tree** that realizes the
workspace → project → section hierarchy (FR-1): nodes with title, Markdown content, and
ordered subnodes; sections as top-level nodes; projects as ordered collections of
sections plus metadata such as status (FR-39); workspaces as collections of projects plus
their common information, which is flexible and follows no predefined structure (FR-43,
FR-5, FR-37). It assigns **stable identity** to workspaces, projects, and
sections only — never to inner nodes — and models structure, never meaning (FR-44). It
enforces **versioning and optimistic concurrency at section granularity** (FR-57–FR-60,
see [§7](#7-cross-cutting-concerns)).
*State.* **Owns the knowledge base** — the authoritative node trees and per-section
version history (the single source of truth, FR-6; permanent, NFR-1). It exposes and
accepts content as the human-readable Markdown/node-tree representation (FR-42), so
hand-edits round-trip losslessly (FR-45).
*Interfaces.* Tool layer and application service (inbound, for agent writes and direct
edits respectively); it notifies the retrieval/index subsystem of changes so the index
stays current.

### Retrieval / index subsystem
*Job.* Make the knowledge base findable at single-user scale (NFR-13). It supports
three complementary access modes — **semantic search** (first-class, since relevant
projects are often unnamed), **keyword search**, and **structured navigation**
(workspace → project → section → nested nodes) — and assembles a **working set of the
relevant sections across the referenced projects**, ranked to fit the model context,
rather than loading whole projects (FR-61). The retrieval agent is its principal
client.
*State.* A derived **semantic index** (and supporting keyword index) over knowledge-base
content. This state is **derived, not authoritative** — it can be rebuilt from the
knowledge base — and is kept current as sections change.
*Interfaces.* Retrieval agent (inbound); knowledge store service (change notifications,
content reads).

### Capture pipeline
*Job.* Take raw input from the front of the funnel to the point where the orchestrator
can act on it (FR-8, FR-48). It accepts typed text, speech (transcribed
**server-side**), documents (text/Markdown, PDF, common office formats, HTML), and web
pages **fetched** from user-provided links and stored as a **snapshot** at ingest time
(FR-48, FR-49). It processes **interactively** for quick typed inputs and
**asynchronously** for heavy inputs — documents, long audio, fetched pages — accepting
them immediately and processing in the background, with results arriving as proposals in
the review queue or auto-applied under autonomy (FR-50). Before writing, capture is
**reconciled against current knowledge-base state** so re-sharing similar content
updates or de-duplicates rather than blindly appends (FR-51).
*State.* None authoritative of its own; it writes raw inputs to the source-artifact
store and hands interpreted results to the orchestrator. Heavy captures rely on a
durable async work mechanism so nothing is lost (NFR-12).
*Interfaces.* Application service (inbound); transcription and link-fetch capabilities;
source-artifact store; orchestrator (to apply interpreted results).

### Source-artifact store
*Job.* Retain raw inputs as **source artifacts** — uploaded files, transcripts,
fetched-page snapshots, audio — linked from the knowledge-base entries they produced,
for provenance (FR-49). Retaining originals is also what makes failed processing
**re-processable** (FR-53, NFR-12).
*State.* **Owns source artifacts.** These are **interaction/source state, not
knowledge** (NFR-11), and are **user-deletable** (FR-49); deleting them never
compromises the knowledge base as the source of truth.
*Interfaces.* Capture pipeline (writes); knowledge store version history and review
queue (reference artifacts by link, FR-58).

### Review queue / effect tracker
*Job.* Hold, durably, the two kinds of pending work the platform must never drop:
**proposed changes awaiting confirmation** under propose-and-confirm autonomy (FR-22,
FR-47) and **failed effects awaiting retry** (FR-53). It tracks the per-effect status of
each input's outcomes so that successes are durable and not rolled back when a sibling
fails (FR-52), and it surfaces every pending or failed item to the user rather than
failing silently (FR-53, FR-56). In-flight clarifications likewise survive sessions and
reconnects here (FR-47).
*State.* **Owns the review queue** — authoritative platform state persisted until each
item is resolved (NFR-11).
*Interfaces.* Tool layer and orchestrator (enqueue proposals/failures, record effect
status); application service (user confirms, retries, or discards items); connector
adapter and knowledge store service (to apply or retry effects).

### Connector adapter
*Job.* Integrate the external task platform through a **pluggable connector** (FR-16),
isolating the rest of the platform from any specific task platform. It owns the
**project↔task mapping** — Personal Brain's authoritative record of which external task
belongs to which project (FR-15) — and, on task creation, asks the connector to place
the task in the platform's native structure corresponding to the project (FR-31). It
performs **reconciliation**: pulling external task changes back into the knowledge base
**when the user interacts** with the affected project, and **sooner via pushed events**
where the connector supports them, without constant polling (FR-30, FR-12). Retries of
task effects are reconciled against the mapping and current state so a partially
succeeded create is not duplicated (FR-55).
*State.* **Owns the project↔task mapping** — authoritative and permanent (FR-15,
NFR-1). Tasks themselves live outside, in the external platform (FR-14).
*Interfaces.* Tool layer (task reads/writes); external task platform via the connector
(outbound); knowledge store service and effect tracker (to reflect task changes back and
to retry failed task effects).

### Model gateway
*Job.* Mediate **all** agent model calls so model sourcing is **selectable per agent**
(FR-20, NFR-7) across third-party hosted APIs and self-hosted open models. It is the
single place that enforces provider terms — third-party providers used only under
**no-training / zero-retention** terms (NFR-10) — and the seam at which the reserved
per-workspace privacy levels (confining sensitive content to self-hosted models) would
later attach (NFR-10).
*State.* None authoritative; reads model-sourcing configuration (account-global today,
NFR-15) and pulls provider credentials from the secret store.
*Interfaces.* Agent runtime (inbound); model providers (outbound); secret store.

### Identity/auth and secret store
*Job.* **Identity/auth** authenticates users via email-based or OAuth sign-in, with
enterprise SSO left open for the future (NFR-8); it establishes the identity and tenancy
boundary that keeps the system collaboration-ready (NFR-5). The **secret store** holds
connector credentials and model API keys in a **managed secret store, never in the
knowledge base** (NFR-9).
*State.* Identity records; secrets. Both authoritative and managed.
*Interfaces.* Application service (auth); connector adapter and model gateway (secrets).

## 5. Data architecture — authoritative state

The architecture turns on a sharp separation between **knowledge** and
**interaction/source state** (NFR-11). The separation is what keeps the
single-source-of-truth principle (FR-6) intact: knowledge has exactly one home, and
everything else is explicitly *not* knowledge.

**Knowledge (the single source of truth, FR-6; permanent, NFR-1)** — owned by the
knowledge store service:
- **Node trees** — the content of every workspace's common information and every
  project's sections, modeled as nodes (title, Markdown content, ordered subnodes) and
  exposed in the human-readable Markdown representation (FR-42, FR-43).
- **Per-section version history** — every change to a section produces a new version,
  recording actor, timestamp, and a link to the originating source artifact or task
  event; this history doubles as the audit trail (FR-57, FR-58, NFR-2).
- **Stable identity** for workspaces, projects, and sections only; inner nodes have no
  durable identity and version as part of their section (FR-44, FR-57).

**Other authoritative platform state (permanent, NFR-1):**
- **Project↔task mapping** — owned by the connector adapter (FR-15).
- **Review queue** — pending proposals and failed effects, owned by the effect tracker
  (FR-47, NFR-11).
- **Settings** — per-tool autonomy, connector configuration, model/privacy sourcing;
  account-global in the current scope (FR-21, NFR-15).

**Interaction/source state (explicitly not knowledge; user-deletable, NFR-11):**
- **Conversation log** — persisted as a log the user can revisit, in a single assistant
  conversation rather than per-project threads; never a knowledge source (FR-46).
- **Source artifacts** — retained raw inputs with provenance links (FR-49).

The **representation is the contract.** Because the knowledge store exposes and accepts
the same Markdown/node-tree form to agents (via tools) and to the structured outline
editor (via direct edits), both paths operate on identical content, and hand-edits
round-trip losslessly (FR-42, FR-45). The platform models the node *tree* but never what
a node *means*: a topic being "open" or "resolved" is content the agents and user
interpret, not a platform-tracked entity, and a task→topic link is a semantic
association optionally reinforced by an in-content reference, never a stored foreign key
(FR-44, FR-12).

The derived **semantic/keyword index** is the one significant piece of **non-**
authoritative state: it can always be rebuilt from the knowledge base, which is why
losing or migrating it is a recoverable operation, not data loss.

## 6. Key flows

These describe, in prose, how the components interact for the spec's supported
interaction patterns. Each is the orchestration control flow (resolve project → select
and invoke agents → combine and apply under autonomy) specialized to a case.

**Typed interactive capture.** The client sends typed text to the application service,
which forwards it to the orchestrator. The orchestrator uses the retrieval agent to
resolve the target project(s) (FR-27, FR-34), invokes the agents needed, and assembles
the intended effects. Each effect passes through the tool layer's autonomy gate: under
default propose-and-confirm it becomes a review-queue proposal (FR-22); under granted
autonomy it applies immediately (FR-23). The first response streams back within a few
seconds (NFR-14).

**Heavy / asynchronous document capture.** The client uploads a document, long audio, or
a link. The application service hands it to the capture pipeline, which stores the raw
input as a source artifact (FR-49), accepts the input immediately, and processes it in
the background — transcribing audio server-side, fetching and snapshotting a linked page
(FR-48, FR-49). Interpreted results return to the orchestrator and arrive as proposals
in the review queue, or auto-apply under autonomy (FR-50). The user is free to move on;
nothing waits on an interactive deadline (NFR-14).

**Multi-project routing.** When an input spans several projects (a status update across
projects), the orchestrator resolves *each* referenced project via the retrieval agent
and produces effects per project, updating each accordingly (FR-10). The retrieval agent
assembles a working set of the relevant sections across those projects rather than
loading them whole (FR-61).

**Retrieval / Q&A.** The user asks for context — a project's status or open points. The
orchestrator routes to the Q&A agent, which (via the retrieval subsystem) gathers a
ranked working set of relevant sections and assembles an answer (FR-13, FR-61). Closed
projects are excluded by default unless explicitly included (FR-40). This is a read; no
effects are produced.

**Direct hand-edit.** The user edits a section in the structured outline editor. The
application service sends the edit straight to the knowledge store service — the
orchestrator and agents are not involved (FR-41). The edit is mediated by the store like
any write (FR-45), produces a new section version attributed to the user (FR-58), and is
immediately visible to agents, which keep no private state (FR-6).

**Task reconciliation.** When a task changes or completes in the external platform, the
connector adapter reflects it back into the knowledge base — for example a Project log
entry or the resolution of an Open topic, realized by an agent editing the relevant node
(FR-12). Reconciliation runs when the user next interacts with the affected project, and
sooner where the connector can push events; it does not rely on constant polling (FR-30).

**Autonomy gate / propose-and-confirm.** Every tool invocation is evaluated at the tool
layer against the per-tool autonomy setting (FR-21). Auto-approved tools run; others are
held as durable proposals the user confirms later — surviving sessions and reconnects,
which is what makes confirm-by-default usable for asynchronous mobile capture (FR-22,
FR-47).

**Concurrency conflict.** Each write carries the section version it was based on. If the
section has moved on, the knowledge store rejects the stale write rather than
overwriting (FR-59): an agent re-reads and re-proposes; a conflicting hand-edit is
flagged to the user. A proposal waiting in the review queue is applied against the
*current* version, so a section edited after the proposal was made is caught (FR-59).
Because the section is the unit, edits to different sections of the same project do not
collide.

**Failure and retry.** Effects apply best-effort, per effect, with tracked status
(FR-52). A failed effect — a knowledge-base write, a task change, or a whole input that
failed to process — becomes a retriable item in the review queue (FR-53); the retained
source artifact lets a failed input be re-processed (FR-49). If the connector is
unavailable, knowledge-base effects still apply and task effects wait as retriable items,
converging via reconciliation (FR-54). Retries are reconciled against the project↔task
mapping and current state so a partially succeeded effect is not duplicated (FR-55). The
net is eventual consistency between the two systems, with every gap visible to the user
(FR-56).

## 7. Cross-cutting concerns

**Reliability and eventual consistency (FR-52–FR-56, NFR-12).** There is no cross-system
transaction. The platform's reliability model is best-effort per-effect application,
durable tracking of every effect's status, retained raw inputs for re-processing, and
the project↔task mapping plus reconciliation as the convergence mechanism. The
architectural consequence is that the **effect tracker / review queue is a first-class
durable component**, not an afterthought, and that no effect path is allowed to fail
silently.

**Versioning and concurrency (FR-57–FR-60).** The **section is the unit of writing,
versioning, and concurrency**. Optimistic concurrency at section granularity keeps the
knowledge base correct under writes from agents, hand-edits, and reconciliation, from
multiple devices, without locking. Full per-section history makes the agent-maintained
base trustworthy: reviewable, recoverable, and an audit trail (NFR-2). Reverting a
section to a prior version is itself a new version (FR-60).

**Security and privacy (NFR-8, NFR-9, NFR-10).** TLS in transit and encryption at rest
(provided by the managed infrastructure); connector credentials and model API keys in a
managed secret store, never in the knowledge base. All model access flows through the
model gateway, which enforces no-training / zero-retention terms for third-party
providers and is the attach point for the reserved per-workspace privacy levels that
would confine sensitive content to self-hosted models.

**Statelessness and scale (FR-6, NFR-6, NFR-13).** Because agents hold no private
knowledge, the compute tier holds no authoritative state and can be scaled and replaced
freely; all authoritative state is in managed storage. The system is sized for
single-user scale (hundreds of projects, thousands of sections), which keeps the
semantic index and working-set assembly bounded.

**Model-agnostic operation (FR-20, NFR-4, NFR-7).** No agent calls a provider directly;
the model gateway is the single indirection, so adding a provider, swapping a model, or
moving an agent to a self-hosted model is a configuration change, not a code change to
the agent.

**Extensibility (NFR-3, FR-35).** Three extension seams are deliberate: task connectors
are pluggable behind the connector adapter; custom sections are user-definable as more
nodes of the same kind, requiring no schema change (FR-4, FR-44); and the agent roster
grows behind the orchestrator without the user defining or composing agents (FR-18,
FR-35). *(How a newly added agent is selected and how its output is integrated remains
an open question — see [§10](#10-open-technical-questions).)*

**Auditability and transparency (NFR-2).** The per-section version history, with actor,
timestamp, and originating-input link, makes "why did this change?" always answerable.
Combined with the single-source-of-truth principle, everything an agent knows is visible
to the user.

## 8. Deployment view (tech-agnostic)

The platform is a cloud-hosted service on managed PaaS infrastructure; users reach it
through thin web and mobile clients, and agents execute server-side (NFR-6). Stated as
**capabilities the deployment must provide** — not products:

- **Stateless compute** for the application service, orchestrator, agent runtime, tool
  layer, and capture workers — horizontally scalable and disposable because they hold no
  authoritative state.
- A **managed datastore** for authoritative state: the knowledge base with per-section
  version history, the project↔task mapping, the review queue, and settings.
- An **object store** for source artifacts (files, transcripts, snapshots, audio).
- A **semantic index** (and keyword index) for retrieval — derived, rebuildable state.
- A **durable async work mechanism** (a queue or equivalent) so heavy captures and
  retries are never lost.
- A **managed secret store** for connector credentials and model API keys.
- **Model hosting** for self-hosted open models, co-located on the same infrastructure,
  reached — like any provider — through the model gateway.
- Platform-provided **TLS and encryption at rest**.

The mapping from these capabilities to concrete services is deferred. The architecture
assumes only their existence, not their identity, so the selection can be made later
without reshaping components.

## 9. Collaboration-readiness (future)

The product is single-user today, but the architecture keeps the door open to multi-user
sharing at project or workspace granularity (NFR-5) without committing to it:

- **Identity and tenancy boundaries are explicit** even though trivial now (one user):
  authentication establishes identity, and authoritative state is reachable through that
  boundary rather than assuming a global single owner.
- **Stable identity on workspaces, projects, and sections** (FR-44) gives the natural
  future units of sharing and access control a durable handle to attach to.
- **Section-granular versioning and optimistic concurrency** (FR-57–FR-59) already
  tolerate concurrent writers from multiple sources and devices — the same mechanism a
  future multi-user setting would rely on.
- **Configuration is account-global today, with per-workspace scoping reserved**
  (NFR-15): autonomy, connector, and model/privacy settings are structured so that
  narrowing their scope later is an extension, not a redesign.

No structural decision in this document assumes a permanently single-user world.

## 10. Open technical questions

Following the spec's discipline, deferred **technical** choices are recorded here rather
than decided silently. These are downstream of (and in addition to) the product-level
open questions in `SPEC.md`.

- **Concrete technology selection.** Every capability in [§8](#8-deployment-view-tech-agnostic)
  — datastore, object store, semantic index, async work mechanism, secret store, model
  hosting — awaits a concrete choice. Deferred deliberately.
- **Index mechanism and freshness.** How the semantic/keyword index is built, embedded,
  and kept current as sections change (synchronous vs. eventual reindexing), and how
  working-set ranking is computed to fit model context (FR-61).
- **Server-side transcription approach.** Which transcription capability is used and how
  it is hosted, given the model-agnostic, optionally self-hosted posture (FR-48, NFR-7).
- **Async work and review-queue semantics.** Delivery guarantees of the async mechanism
  (at-least-once vs. exactly-once), and how the effect tracker deduplicates retries
  against the project↔task mapping in practice (FR-52, FR-55).
- **Snapshot format.** How a fetched web page is captured as a faithful, durable snapshot
  at ingest time (FR-49).
- **Push-event transport.** How connectors deliver task-change events where supported,
  and the fallback for connectors that cannot push (FR-30) — tied to the spec's open
  **task connector capability contract**.
- **New-agent integration mechanics.** How an added specialized agent is registered,
  selected by the orchestrator, and how its output is combined (FR-35) — the technical
  side of the spec's open **future additions to the roster**.
- **Markdown↔node-tree mapping rules.** The precise, lossless mapping between the stored
  node tree and its Markdown view that guarantees hand-edit round-tripping (FR-42,
  FR-45).
