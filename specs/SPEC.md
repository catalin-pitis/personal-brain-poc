# Personal Brain — Specification

> Living specification for the Personal Brain platform. This document evolves as
> requirements are discovered and refined. See [`CLAUDE.md`](../CLAUDE.md) for how
> we work on it. For **how** the platform is built to satisfy these requirements,
> see the companion [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Overview

Personal Brain helps a user centralize and structure information about their
projects, and organize and plan their tasks.

Its defining characteristic is that it is **agent-mediated**. Rather than the
user manually structuring information, the user communicates in natural language —
typed or spoken — and by sharing documents, and an agent maintains a structured
knowledge base on their behalf. The intent is to shift the user's time away from
organizing information and toward thinking and making decisions based on a
knowledge base the agent keeps current for them.

Two jobs sit at the core of the product:

1. **Knowledge** — capturing and structuring information about projects into a
   knowledge base.
2. **Planning** — organizing tasks and planning the work.

These are linked: the planning is about the tasks that advance the same projects
the knowledge base describes.

## Core concepts

_The shared vocabulary of the product. Definitions are filled in as they are
decided; terms still under discussion are marked as open._

- **Knowledge base** — the persistent, structured store of information about the
  user's projects; the platform **owns and hides its underlying storage form**,
  exposing the knowledge base in a stable, human-readable representation (see
  Knowledge base representation). It is the **single source of truth**: agents read from it and
  keep no private state of their own, so everything an agent "knows" is visible to
  the user. Agent-driven changes are applied through the orchestrator (see
  Orchestration); the user can also read **and edit** the knowledge base
  directly, by hand, without an agent in the loop. (Sharing project
  information between multiple users is a planned future direction, not part of
  the current scope.)
- **Agent** — an actor that interprets what the user communicates, decides what
  it pertains to, and produces the resulting updates, which the orchestrator
  applies to the knowledge base. The platform provides **multiple specialized
  agents** rather than one; agents are the primary means of getting information
  *into* and *organized within* the knowledge base.
- **Orchestration** — the platform's coordination of its specialized agents,
  performed by a single **orchestrator** that is the entry point for all user
  input. It resolves which project(s) an input concerns, selects and invokes the
  agents needed, combines their work, and applies the result under the configured
  autonomy. When it cannot determine the target project confidently, it asks the
  user.
- **Workspace** — a **logical grouping** of projects that the user defines (for
  example, by client or by theme), and the top-level container in the knowledge
  base. A workspace may hold information common to all the projects within it (for
  example, contact persons and their roles), so shared context is not repeated in
  each project.
- **Project** — a larger, longer-lived effort the user works toward, and the
  primary unit around which information is organized. Every project belongs to a
  workspace. A project has a **status** (for example, *preparing*, *active*, or
  *closed*) that, among other things, scopes which projects normal requests
  consider.
- **Node** — the generic unit of project information: a **title**, **content**
  (human-readable Markdown), and an ordered list of **subnodes**. Nodes nest to
  arbitrary depth, forming a tree. The platform models the node *tree* — titles,
  content, nesting, ordering — but **not what a node means**: whether a node is a
  problem, a decision, or an open topic is **interpreted by the agents** from its
  title, content, and position, not tracked by the platform (see Knowledge base
  representation).
- **Section** — a **top-level node** under a project, and the unit the platform
  gives **stable identity** and manages change at (versioning, concurrency). A
  project is an ordered collection of sections; each section is a node tree whose
  nested structure is content the agent and user maintain. The built-in sections
  (Open topics, Decisions, Project log) are sections the platform seeds by default.
- **Task** — a discrete unit of work the user does. Every task belongs to a
  project. Tasks are **not** stored in the knowledge base; they are managed in an
  external task-management platform that Personal Brain integrates with. Tasks are
  what the planning side of the product organizes.
- **Task-management platform** — the external system of record for tasks.
  Personal Brain does not store tasks itself; it reads and updates them through a
  configurable **connector**. The specific platform is not fixed: the user (or,
  in a future multi-user setting, an administrator) configures a connector to
  whatever task-management platform they use, so the integration is pluggable (an
  MCP connector is the intended mechanism). Agents act on tasks there on the
  user's behalf.
- **Project↔task mapping** — Personal Brain's own internal, authoritative record
  of which external task belongs to which project. Keeping the association inside
  Personal Brain makes the link robust and uniform across different task
  platforms. For placing and organizing tasks *within* the external platform, the
  connector additionally maps a Personal Brain project to that platform's native
  structure — its own projects or sections.
- **Tool** — an operation, provided through **MCP**, that **reads or writes
  data**. Tools span both the external task platform (creating, updating, and
  querying tasks) and the knowledge base (reading and writing project and
  workspace information). Tools are the unit at which **autonomy** is configured:
  the user can let some tools run automatically — for example, read-only ones —
  while others still require confirmation. Autonomy is set **per tool** and
  applies uniformly, regardless of which agent invokes the tool.
- **Source artifact** — a raw input retained for provenance: an uploaded file, a
  transcript, an audio recording, or a snapshot of a fetched web page, linked from
  the knowledge-base entries it produced. Source artifacts are **interaction/source
  state, not knowledge**, and are user-deletable (see Capture pipeline).
- **Review queue** — the durable store of agent-proposed changes awaiting the user's
  confirmation (under propose-and-confirm autonomy), together with failed effects
  awaiting retry. It is authoritative state the platform persists until each item is
  resolved (see Conversation and interaction state; Reliability and consistency).

## Workspaces and projects

Projects are grouped into **workspaces**. A workspace is the top-level container
in the knowledge base, and every project belongs to one. The hierarchy is
therefore *workspace → project → project information*.

A workspace is a **logical grouping the user chooses**; the platform does not
dictate the grouping criterion. For example, a workspace might represent a
**client**, with the various projects and initiatives for that client grouped
inside it; or it might represent a **theme** such as *investments*, where each
project corresponds to an individual investment. The grouping exists both to share
context across related projects and to keep them organized together.

A workspace may also hold information **common to all its projects** — for
example, the contact persons involved and their roles — so that shared context
does not have to be repeated in each project. This workspace-level common
information is **stored and maintained just like project information** — it lives
in the knowledge base, agents maintain it, and the user can read and edit it
directly — but, unlike a project, it does **not follow a predefined structure**:
it does not start from the built-in sections and holds whatever shared context is
relevant, in a flexible form. Within a workspace, each project then carries its
own information, structured as described below.

Workspaces — and the projects within them — can be **created and managed by
chatting with the agent**, consistent with the agent-mediated model: the user asks
the agent to set up or reorganize the grouping. The user can **also edit the
structure directly by hand** — both paths are supported and operate on the same
knowledge base.

## Project information

Information about a project is organized into **named sections**. A project
starts from a **minimal built-in structure** and can be **customized**: the user
can add sections to track whatever matters for that project. The structure is not
fixed by the platform beyond a small default. Customization is **per individual
project** — each project's sections are its own. _(Reusable **project templates**,
selectable when a project is created, are a possible future direction.)_

The minimal built-in sections are:

- **Open topics** — things currently open or unresolved on the project.
- **Decisions** — a record of decisions made, kept for later reference.
- **Project log** — a chronological history of what has happened on the project,
  for reference.

Beyond these, the user can add their own sections — for example KPIs, a risk
list, or anything else worth tracking about the project. (This is distinct from a
project's lifecycle status, described below.)

Sections are **not flat**. Within a section, information nests as **subnodes** to
whatever depth is useful — a problem can carry its own sub-topics, a decision its
own rationale and follow-ups. The platform models this nesting (titles, content,
ordering); what each node *means* is left to its content and the agents'
interpretation (see Knowledge base representation).

The agents maintain these sections as information comes in. Because the structure
(including any custom sections) is part of the knowledge base, the user can read
**and edit** it directly as well, by hand.

Each project also has a **status** — for example *preparing*, *active*, or
*closed*. Status affects scope: by default, **normal requests exclude closed
projects** (for instance, a cross-project status query or the routing of new input
considers only active and preparing projects), while the user can still include
closed projects explicitly when needed.

## Knowledge base representation

The platform **owns and hides** how the knowledge base is stored. Whatever the
underlying store, the platform exposes — and accepts — knowledge in a stable,
**human-readable representation**: content is expressed in **Markdown**, which is
readable, hand-editable, portable, and able to carry structure by convention
(headings, lists, checkboxes) without imposing a rigid schema.

This separates two kinds of mediation that the spec previously blurred:

- **Storage mediation is always present.** Neither the user nor the agents touch
  the underlying store; everything goes through the platform.
- **Agent mediation is optional.** Agent-maintained updates and direct
  hand-editing both go through the platform and operate on the same human-readable
  content; the difference is only whether an agent is in the loop.

### Content model: nodes

Information is modeled as a **tree of nodes**. A **node** has a **title**,
**content** (human-readable Markdown), and an ordered list of **subnodes**; nodes
nest to arbitrary depth. A node tree and a Markdown document are two views of the
same thing — headings give titles and nesting, body text gives content — so the
human-readable representation (FR-42) and lossless hand-editing (FR-45) are
preserved.

- A **section is a top-level node** under a project.
- A **project is an ordered collection of sections**, plus project-level metadata
  such as its status. Each section is a node tree.
- A **workspace is a collection of projects**, plus its common information — itself
  node(s) in the flexible, no-predefined-structure form (FR-37).
- The built-in sections (Open topics, Decisions, Project log) are **not special**:
  they are sections the platform **seeds by default**, and custom sections are more
  sections of the same kind.

This draws one deliberate line:

> **The node tree — titles, content, nesting, ordering — is the platform's
> concern; what a node *means* is content the agents and user interpret.**

The platform knows about workspaces, projects, and the node tree within each
project, and it gives **stable identity** to workspaces, projects, and **sections**
(the top-level nodes) — projects in particular need it for the project↔task mapping.
It does **not** assign identity to nodes *below* the section, and it does **not**
model what any node means. "Open topics" is a section whose subnodes are individual
topics; the agent and the user maintain that internal structure — including whether
a topic is open or resolved — as **content**, not as platform-tracked entities with
their own state.

Two consequences follow:

- **Resolving an Open topic** (FR-12) means the responsible agent **edits the
  relevant node**, not a state transition on a tracked entity. The link from an
  external task back to the topic it originated from is a **semantic association the
  agent re-derives**, optionally reinforced by a lightweight reference the agent
  writes **into the node content** itself — not a stored foreign key. (Inner nodes
  have no durable identity, so a stored reference would not be stable anyway.)
- **Hand-editing round-trips losslessly.** A section's content *is* its node tree,
  edited directly through a **structured outline editor** (or as the equivalent
  Markdown); what the user edits is exactly what is stored and shown, with no lossy
  translation into sub-entities.

## History, versioning, and concurrency

The knowledge base is written by more than one party — agents (via the
orchestrator), direct hand-edits, and reconciliation from task events — and from
more than one device. The **section is the unit of writing**: a change anywhere in a
section's node tree is a change to that section, and the platform manages change at
that granularity. (Inner nodes have no independent identity or history; they live
and version as part of their section.)

**Versioning.** Every change to a section produces a **new version**, and the
platform **keeps the full version history** of each section. History is what makes an
agent-maintained knowledge base trustworthy: changes are reviewable and recoverable.

**Audit trail.** Each version records the **actor** (the user, a specific agent, or
reconciliation), a **timestamp**, and a **link to what caused it** — the originating
source artifact (FR-49) or task event. The history therefore doubles as an audit
trail: "why did this change?" is always answerable, reinforcing transparency
(NFR-2).

**Concurrency.** Writes use **optimistic concurrency** at section granularity. A
write states the section version it was based on; if the section has since moved on,
the stale write is **rejected and re-driven** rather than silently overwriting — an
agent re-reads and re-proposes, and a conflicting hand-edit is flagged to the user. A
proposal waiting in the review queue (FR-47) is likewise applied against the
**current** version, so a section edited after the proposal was made is caught.
Because the section — not the whole project — is the unit, edits to different
sections of the same project do not collide.

**Undo.** The user can **revert a section to a prior version**; the revert is itself
recorded as a new version.

## How it works: agents and the knowledge base

Agents are the primary means of **maintaining** the knowledge base. The user
supplies information in natural language or as documents; an agent interprets it,
determines what it pertains to, and updates the relevant parts of the knowledge
base. The user does not have to structure the information themselves.

The platform provides **multiple specialized agents**, each competent in a
different area, and may use a different underlying model for each. When the user
provides input, the platform's **orchestration** routes it to the agent(s) best
suited to handle it and combines their work when the input spans more than one
concern or project (for example, a status update touching several projects).

Agents are specialized along two axes, which can combine:

- **By kind of work** — agents that carry out the platform's core mechanisms,
  such as **retrieval** and **Q&A** agents that underpin knowledge update and
  processing.
- **By subject/domain** — agents with expertise in a particular domain, such as a
  **finance agent** that computes KPIs to include in a project's information.

A **finance agent** (domain) and a **planning agent** for organizing tasks
(kind of work) are illustrative examples of further specialized agents that could
be orchestrated; they are not committed components, and their capabilities are not
specified in this iteration.

Agents are **not** specialized per project; no agent is tied to an individual
project. Orchestration draws on the relevant work-type and domain agents to
handle a given input.

Everything the user **directs to the agents** flows through a single
**orchestrator** — the entry point for all input the user types, speaks, or
uploads for the agents to act on; there is no direct path to a specific agent.
(Editing the knowledge base directly by hand is a separate path that does not
involve the orchestrator.) For each input the orchestrator:

1. **Resolves the target project(s).** The user may name the project in their
   message; otherwise the orchestrator infers it from the content. If it cannot
   determine the project confidently, it **asks the user** rather than guessing.
2. **Selects and invokes the agents.** It determines which work-type and domain
   agents the input requires — for example, pulling in the finance agent when a
   KPI must be computed — and runs them.
3. **Combines and applies the results.** It assembles the agents' outputs into
   knowledge-base updates and task changes, applying them under the configured
   autonomy (confirm by default).

Specialized agents **return their results to the orchestrator**, which applies
them to the knowledge base; agents do not write to the knowledge base
independently. This keeps routing, combination, and the autonomy gate all in one
place.

Agents are **defined as part of the product** during development. Normal users
do not define, configure, or compose agents; they interact with the platform and
the orchestration selects the right agents on their behalf.

The knowledge base holds **project information** and is the single source of
truth for it. **Tasks live outside** the knowledge base, in an external
task-management platform. Both are accessed through **tools** (via MCP): agents
read through them, and the orchestrator applies writes under the configured
autonomy. Agents act on tasks in the external platform rather than storing them
locally. Every task belongs to a project, and Personal Brain records this in its
own internal **project↔task mapping** rather than relying on the external
platform's
structure, so the link holds across whatever platform is connected.

Capture crosses the knowledge/task boundary in **both directions**. A single
input — for example shared meeting notes — may **both** update project
information **and** create or update tasks in the external platform: conclusions
update the knowledge base while action items become tasks. Conversely, when a
task changes or is completed in the external platform, that change is **reflected
back** into the knowledge base (for example as a Project log entry, or by
resolving an Open topic). Personal Brain reconciles such changes when the user
next interacts with the affected project, and sooner where the connector can push
change events.

How much the agents act on their own is **configurable**. By default, an agent
**proposes** its changes and the user must **confirm** them before they take
effect. The user can choose to grant more autonomy — allowing changes to apply
automatically, either fully or for **specific tools only**. For example, the user
might auto-approve tools that merely *read* data while still confirming tools that
*write* (create or modify knowledge or tasks). Autonomy is configured **per tool**
and applies uniformly, regardless of which agent invokes the tool — it is not
scoped per individual agent.

The user can communicate with the agents by **typing or by speaking**. Spoken
input is supported as a first-class way to interact, because free-form speech is
often a faster and more natural way to capture what happened than writing it out.
Voice is supported for **input only**; the agents respond in text, not by voice.

Importantly, the agents are the **primary** way to maintain the knowledge base,
but not the only one: the user can always **read and edit** the project
information directly, by hand — editing the human-readable representation through a
platform surface, **not** by touching the underlying store, which the platform
always mediates. This makes *agent mediation* optional while *storage mediation* is
always present. Agent-mediated maintenance and direct editing operate on the same
single source of truth, so hand edits are immediately visible to the agents (which
keep no private state).

### The agent roster

The committed core is deliberately **small**:

- **Orchestrator** — the sole entry point; resolves the target project, selects
  and invokes agents, combines their results, and applies changes under the
  configured autonomy.
- **Retrieval agent** — finds the relevant project(s) and existing information in
  the knowledge base. The orchestrator relies on it to identify which project an
  input concerns.
- **Q&A agent** — assembles answers and contextual information from the knowledge
  base in response to the user's questions.

**Extraction** (turning raw notes or transcripts into structured updates and
spotting action items) and **writing** (deciding what to change in which section)
are treated as **capabilities the orchestrator coordinates**, not as separate
agents.

Beyond this core, the roster can grow with **additional specialized agents** — by
subject/domain (for example, a finance agent) or by kind of work (for example, a
planning agent) — defined as part of the product when needed. These additions are
not part of the committed core.

### Supported interaction patterns

These scenarios illustrate the intended behavior. They are decided in spirit;
exact mechanics are refined below and in Requirements as they are worked out.

1. **Conversational capture.** The user describes what happened in their own
   words (for example, the conclusions of a project meeting) in the chat, and
   the agent updates the information related to that project to reflect it.

2. **Document capture.** The user shares notes or a meeting transcript (their
   own notes or a transcription), and the agent extracts the relevant
   information and updates the project's information. The same input may also
   create or update tasks — for example, action items in the notes become tasks
   in the external platform while the conclusions update the knowledge base.

3. **Multi-project routing.** The user shares the conclusions of a status
   meeting that spans several projects. The platform's orchestration identifies
   each project the input refers to and updates the related information for each
   one, drawing on the appropriate agent(s).

4. **Contextual retrieval.** When preparing for a meeting or a new piece of work, the
   user asks the agent for context — for example, the status of a project or its
   open points — and the agent provides the requested details.

5. **Direct access.** The user reads — or edits — the project information
   directly, by hand, without the agent in the loop.

6. **Structural management.** The user creates or reorganizes workspaces and
   projects by asking the agent — for example, setting up a new client workspace,
   opening a project within it, or marking a project as closed.

## Capture pipeline

This is the path from raw input to knowledge-base and task updates. Its control
flow is the orchestration already described (resolve project → select and invoke
agents → combine and apply under autonomy); this section fixes the **front of the
pipeline** and its **runtime characteristics**.

**Accepted inputs.** The user can provide:

- **Typed text** in the conversation.
- **Speech**, transcribed **server-side** (keeping clients thin, NFR-6, and the
  transcription model swappable/self-hostable, NFR-7); voice is input only (FR-36).
- **Documents** — plain text and Markdown, PDF, common office documents, and
  **HTML**.
- **Web pages pointed to by links** — when the user provides a URL, the platform
  **fetches** the page and ingests its content like any other document.
- *(Images / OCR are a possible later extension.)*

Concrete size limits are platform-defined and tunable.

**Raw input retention.** Originals are **retained as source artifacts** — the
uploaded file, the transcript, the fetched web page (stored as a **snapshot** taken
at ingest time, since the live page may change), and the audio — and are **linked
from the knowledge-base entries they produced**, for provenance. Source artifacts
are **interaction/source state, not knowledge** (the same distinction drawn for the
conversation log): they do not compromise the knowledge base as the single source
of truth, and the user can **delete** them. (Retention windows are tunable; see Open
questions.)

**Processing model.** Capture is handled **interactively** for quick typed inputs,
and **asynchronously** for larger captures (documents, long audio, fetched pages):
the input is accepted immediately and processed in the background, with results
arriving as proposals in the **durable review queue** (or auto-applied under
autonomy). This makes capture-and-go practical on mobile.

**Duplicate handling.** Before writing, capture is **reconciled against the current
knowledge base** (agents read before they write): re-sharing similar content updates
or de-duplicates rather than blindly appending. Duplicate-resistance is a property
of the extract/write step, not input-level idempotency.

## Reliability and consistency

A single input can produce **multiple effects across two systems** — knowledge-base
section writes (a store the platform controls) and task changes (an external platform
it does not). True cross-system atomicity is not achievable, so the platform applies
effects on a **best-effort, per-effect basis** with **tracked status**:

- **Each effect is applied independently.** A knowledge-base section write or a task
  change succeeds or fails on its own; successes are durable and are **not** rolled
  back if a sibling effect fails. There is no all-or-nothing guarantee — across the
  two systems or even within the knowledge base.
- **No silent failures.** Any effect that fails to apply — and any input that fails
  to process — becomes a **retriable item** surfaced to the user. This reuses the
  **durable review queue** (FR-47), which holds pending *and* failed effects awaiting
  retry.
- **Inputs are never lost.** Because raw inputs are retained as source artifacts
  (FR-49), a processing failure (an agent/model error, a failed link fetch) can be
  **re-processed**; nothing is left half-interpreted.

**Component unavailability** is handled the same way:

- *Connector unavailable* — knowledge-base effects still apply; task effects wait as
  retriable items, and **reconciliation (FR-30)** catches up later.
- *Model/agent failure mid-processing* — the failure is reported and the retained
  input can be re-processed.
- *Link fetch failure* — that input fails cleanly and is reported.

**Retry is reconciled.** Retrying a failed task effect is checked against the
**project↔task mapping** and current state (FR-51) so a partially-succeeded create is
not duplicated.

The net model is **eventual consistency** between the knowledge base and the external
task platform: a knowledge-base entry may briefly reference a task not yet created,
with the **mapping and reconciliation** as the convergence mechanism and every gap
**visible** to the user rather than silent.

## Conversation and interaction state

The product is used through a conversation, so the platform keeps **interaction
state** — but this is deliberately distinct from the **knowledge** in the knowledge
base, and that distinction is what keeps the single-source-of-truth principle
(FR-6) intact.

- **Knowledge** — project and workspace facts — lives in the **knowledge base**,
  the single source of truth. Anything worth remembering about a project is
  distilled into it.
- **Interaction state** — the conversation transcript and the orchestrator's
  in-flight state (a pending clarification, or a proposed change awaiting
  confirmation) — is a **separate concern**, not knowledge.

So "agents keep no private state" (FR-6) is about *knowledge*: there is no project
fact an agent holds that the knowledge base does not. A conversation log is not
private agent memory — it is the user's own chat, fully visible (consistent with
NFR-2), and it is not where project facts live. Agents stay **grounded in the
knowledge base**; recent conversation serves only as **immediate context** within
an exchange, never as a long-term memory the knowledge base does not also hold.

Three decisions follow:

- **Conversation history is persisted** across sessions, as a **log** the user can
  revisit — explicitly **not** a knowledge source. (Its **retention window** is part
  of the open retention question; see Open questions.)
- Interaction happens in a **single assistant conversation**, not per-project
  threads: a single input routinely spans several projects (FR-10), while
  per-project knowledge is found in the knowledge base, organized by project.
  (Threads remain a possible future refinement.)
- Proposed changes awaiting confirmation form a **durable review queue**. Under the
  default propose-and-confirm autonomy (FR-22), a change the user does not confirm
  immediately **survives** for review later, and an in-flight clarification likewise
  survives a dropped connection — which is what makes confirm-by-default usable for
  asynchronous, mobile capture. It is **authoritative state the platform persists**
  (alongside the knowledge base and the project↔task mapping): a store of pending
  proposals, kept **durably until resolved**.

## Deployment and infrastructure

Personal Brain is delivered as a **cloud-hosted service**. The backend, its
databases, and any **self-hosted open-source models** run on **managed
platform-as-a-service (PaaS)** infrastructure; users reach the product through
**thin clients** (web and mobile) rather than running it themselves, and the
agents execute **server-side**, not on the client.

This follows from decisions already made: voice capture is first-class and meant
to be used in the moment (pointing to mobile), the same knowledge base should be
reachable from multiple devices, and the design must not preclude later multi-user
collaboration (NFR-5) — all of which favor a hosted backend over a local-first or
user-self-hosted deployment.

**Model sourcing is flexible.** Consistent with the model-agnostic principle
(FR-20, NFR-4), an agent may be powered either by a **third-party hosted model
API** or by an **open-source model the platform self-hosts** on the same PaaS
infrastructure, and different agents may choose differently. Self-hosting open
models is the lever for keeping sensitive inference off third-party providers; the
privacy posture that governs this is set out under Identity, authentication, and
data handling.

Because the underlying store is a platform-managed concern (FR-42) and agents hold
no private state (FR-6), the hosted services can remain effectively stateless, with
all authoritative state in managed storage — consistent with the
single-source-of-truth model.

## Identity, authentication, and data handling

**Authentication.** Users sign in with **email-based** credentials or via **OAuth**
(social) sign-in; both are supported. Enterprise **SSO** is left open as a later
option for when collaboration arrives.

**Privacy posture (baseline).** Agents may use **third-party hosted model APIs** for
any work, used only under **no-training / zero-retention** terms. Self-hosting
open-source models (NFR-7) is an **optimization**, not a requirement. The
architecture is kept ready — but not yet built — for **per-workspace/project privacy
levels** that would confine sensitive content to self-hosted models; this rests on
the per-agent model sourcing already in NFR-7.

**Core safeguards.**

- **TLS in transit** and **encryption at rest** (handled by the managed PaaS).
- Connector credentials and model API keys live in a **managed secret store**,
  **never in the knowledge base**.
- Any third-party model provider is used only under **no-training / zero-retention**
  terms.

**Retention.** The knowledge base persists permanently until the user changes or
deletes it (NFR-1). Raw inputs are retained as source artifacts (FR-49) and
conversation history as a log (FR-46), both user-deletable; the remaining open item
is the **retention window / auto-purge policy** (see Open questions).

## Scale, performance, and configuration

**Scale and retrieval.** The product is sized for **single-user scale** — on the
order of **hundreds of projects** and **thousands of sections**, each section a node
tree from a paragraph to a few pages. A single project's information typically fits
in a model's context, but a single input or conversation routinely references
**several projects** (FR-10), so retrieval does **not** load whole projects: the
**retrieval agent assembles a working set of the relevant sections across the
identified projects**, ranked to fit the context. Because the relevant projects are not always named,
**semantic search across the knowledge base is a first-class retrieval mechanism** —
alongside structured navigation (workspace → project → section) and keyword search —
not merely an optional enhancement, though it stays bounded by the single-user scale
above.

**Performance.** Targets are **soft guidelines, not SLAs**: interactive operations
(typed capture, retrieval, Q&A) feel **conversational**, a response beginning within
a few seconds; **heavy captures** (documents, long audio, fetched pages) run
**asynchronously** with no interactive deadline, surfacing results in the review
queue when ready. The platform **favors responsiveness and never losing work** over
instant completion.

**Configuration scope.** In the current single-user scope, configuration is
**account-global**: autonomy is set **globally per tool** (FR-21), and the task
connector and privacy/model-sourcing settings are likewise account-level.
**Per-workspace scoping** of any of these is a **reserved future extension**, aligned
with the open questions on multiple task platforms and per-workspace privacy levels.
Settings persist as authoritative platform state in managed storage.

## Requirements

> A first pass derived from the decisions above. Requirements are stated only
> where decided; where an open question blocks a detail, it is noted inline.
> The `FR-`/`NFR-` identifiers are stable handles for reference as the spec
> evolves.

### Knowledge base and structure

- **FR-1** The platform organizes knowledge as a hierarchy of **workspaces →
  projects → per-project named sections**.
- **FR-2** Every project belongs to **exactly one** workspace; a workspace may
  contain many projects.
- **FR-3** Each project starts from a minimal built-in structure consisting of
  the sections **Open topics**, **Decisions**, and **Project log**.
- **FR-4** The user can add **custom sections** to a project to track additional
  information (for example, KPIs or a risk list). Customization is **per
  individual project**; reusable templates are a future direction.
- **FR-5** A workspace can hold information **common to all its projects** (for
  example, contact persons and their roles).
- **FR-6** The knowledge base is the **single source of truth** for project and
  workspace information; agents retain no private state outside it.
- **FR-7** The user can **read** project and workspace information directly,
  without an agent in the loop.
- **FR-37** Workspace-level common information is stored and maintained the
  **same way as project information** (in the knowledge base; agent-maintained and
  directly editable), but has **no predefined structure** — it does not start from
  the built-in project sections and holds shared context in a flexible form.
- **FR-38** Workspaces and projects can be **created and managed
  conversationally**, by chatting with the agent (and also edited directly by
  hand; see FR-41).
- **FR-39** Each project has a **status** (for example, *preparing*, *active*,
  *closed*).
- **FR-41** The user can also **edit** the knowledge base **directly, by hand** —
  project and workspace information as well as the structure itself (sections,
  projects, workspaces) — without an agent in the loop. Agent-mediated maintenance
  (FR-9) and direct editing both operate on the same single source of truth, so
  each path immediately sees the other's changes.
- **FR-42** The platform **owns and hides the underlying storage form** of the
  knowledge base, exposing and accepting knowledge in a stable, **human-readable
  representation (Markdown)**.
- **FR-43** Project information is modeled as a **tree of nodes**, where a **node**
  has a **title**, **content** (human-readable Markdown), and ordered **subnodes**.
  A **section is a top-level node** under a project; a **project is an ordered
  collection of sections** plus project-level metadata (such as status); a
  **workspace is a collection of projects** plus common-information node(s). The
  built-in sections are sections the platform **seeds by default**.
- **FR-44** The platform models the **node tree** — titles, content, nesting, and
  ordering — and gives **stable identity** to workspaces, projects, and **sections**
  (the top-level nodes; projects in particular need it for the project↔task
  mapping). It does **not** give identity to nodes **below** the section, and it
  does **not** model what a node **means**: a node's meaning (problem, decision,
  open topic) and any state (such as a topic being open or resolved) are **content**
  the agents and user interpret, **not** platform-tracked entities.
- **FR-45** The platform **always mediates the underlying store** (neither the user
  nor the agents access it directly), while the **agent is optional in the loop**:
  direct hand-editing operates on the node tree through a platform surface — a
  **structured outline editor** or the equivalent Markdown — and **round-trips
  losslessly** (refines FR-41).

### History, versioning, and concurrency

- **FR-57** Every change to a **section** produces a **new version**; the platform
  keeps the **full per-section version history**. A section versions as a whole,
  including its nested node tree; inner nodes have no independent history.
- **FR-58** Each version records the **actor** (the user, a specific agent, or
  reconciliation), a **timestamp**, and a **link to the originating input** (source
  artifact, FR-49) or task event; the history serves as the **audit trail** (NFR-2).
- **FR-59** Writes use **optimistic concurrency at section granularity**: a write
  carries the base section version, and a **stale write is rejected and re-driven**
  (the agent re-reads and re-proposes; a conflicting hand-edit is flagged) rather
  than silently overwriting. Edits to different sections of the same project do not
  collide. Queued proposals (FR-47) are applied against the **current** version.
- **FR-60** The user can **revert a section to a prior version**; the revert is
  recorded as a new version.

### Capture

- **FR-8** The user can provide information by **typing**, by **speaking**, by
  **sharing documents** (for example, notes or meeting transcripts), or by
  **providing links to web pages**. Accepted formats are detailed in FR-48.
- **FR-9** The platform **interprets** user input and updates the relevant
  information without requiring the user to structure it manually.
- **FR-10** When an input spans **several projects**, the platform routes it to
  each relevant project and updates each accordingly.
- **FR-11** A single input can produce **multiple effects** — updating knowledge
  and creating or updating tasks — as warranted (for example, action items become
  tasks while conclusions update the knowledge base).
- **FR-12** Changes to a task in the external platform (for example, completion)
  are **reflected back** into the knowledge base (for example, a Project log entry
  or resolution of an Open topic). See FR-30 for timing. Resolving an Open topic is
  realized by the agent **editing the relevant node** (FR-44); the task→topic link
  is a **semantic association** (optionally reinforced by an in-content reference),
  not a stored foreign key.
- **FR-36** Voice is supported for **input only**; agents respond in text, not by
  voice.
- **FR-48** Accepted inputs are **typed text**, **speech** (transcribed
  server-side), **documents** (plain text/Markdown, PDF, common office formats, and
  HTML), and **web pages fetched from user-provided links**. Size limits are
  platform-defined; images/OCR are a future extension.
- **FR-49** Raw inputs are **retained as source artifacts** (uploaded files,
  transcripts, fetched-page snapshots, audio), **linked from the knowledge-base
  entries they produced** for provenance, classified as **interaction/source state**
  (not knowledge), and **user-deletable**. A fetched web page is stored as a
  **snapshot taken at ingest time**.
- **FR-50** Capture is processed **interactively** for quick typed inputs and
  **asynchronously** for larger inputs; asynchronous results are delivered as
  proposals to the **durable review queue** (FR-47) or applied under autonomy.
- **FR-51** Before writing, capture is **reconciled against current knowledge-base
  state** to avoid duplication (agents read before writing); the platform does not
  rely on input-level idempotency.

### Retrieval

- **FR-13** The user can ask an agent for **contextual information** about a
  project (for example, its status or open points) and receive the requested
  details.
- **FR-40** By default, **normal requests exclude closed projects** from their
  scope; the user can include them explicitly when needed.
- **FR-61** When resolving inputs and answering questions, the **retrieval agent
  assembles a working set of the relevant sections across the project(s) an input
  references** (FR-10) rather than loading whole projects, using **semantic search**,
  structured navigation (workspace → project → section → nested nodes), and keyword
  search, and **ranking results to fit the model context**.

### Tasks and planning

- **FR-14** Tasks are stored in an **external task-management platform**, not in
  the knowledge base.
- **FR-15** Every task is **associated with a project**. Personal Brain maintains
  this association in its own internal **project↔task mapping**, independent of
  the external platform's organizational features. See FR-30 for sync timing.
- **FR-16** The platform integrates with the task-management platform through a
  **configurable, pluggable connector** (MCP is the intended mechanism). The
  current scope assumes a **single connected task platform**; supporting more than
  one is an open question. _(Minimum connector capabilities are open.)_
- **FR-17** Organizing tasks across projects remains an intended capability,
  expected to be delivered by a specialized agent. Its specifics are **deferred**
  in this iteration; the planning agent stands only as an example of an
  orchestrated specialized agent, not a committed component.
- **FR-30** Personal Brain reconciles external task changes into the knowledge
  base **when the user interacts** with the affected project (baseline), and
  **sooner via pushed events** where the connector supports them. It does not rely
  on constant polling.
- **FR-31** When a task is created for a project, the **connector places it in the
  external platform's native structure** (project or section) corresponding to
  that Personal Brain project. Mapping the project to the platform's structures is
  the connector's responsibility.

### Agents and orchestration

- **FR-18** The platform provides **multiple specialized agents**, defined as part
  of the product. End users do not define, configure, or compose agents.
- **FR-19** The platform **orchestrates** its agents — selecting the appropriate
  agent(s) for an input and combining their outputs. _(The orchestrator model is
  specified in FR-26–29; the project-inference confidence threshold remains
  open.)_
- **FR-20** Different agents may be powered by **different underlying models**.
- **FR-24** Agents are specialized by **kind of work** (for example, retrieval,
  Q&A, planning) and by **subject/domain** (for example, a finance agent that
  computes KPIs); the two axes may combine.
- **FR-25** Agents are **not** specialized per project; no agent is tied to an
  individual project.
- **FR-26** All input the user **directs to the agents** flows through a single
  **orchestrator** — the entry point for typed, spoken, and uploaded input. There
  is no direct path to a specific agent. (Direct hand-editing of the knowledge
  base is a separate path; see FR-41.)
- **FR-27** The orchestrator resolves the target project(s) for an input from the
  user's explicit indication or by inference; if it cannot determine the project
  confidently, it **asks the user** rather than guessing.
- **FR-28** The orchestrator selects and invokes the agents an input requires,
  combines their outputs, and applies the resulting changes under the configured
  autonomy.
- **FR-29** Agents return their results to the orchestrator, which applies the
  changes to the knowledge base; no agent writes to the knowledge base
  independently of the orchestrator.
- **FR-32** The committed core roster consists of the **orchestrator**, a
  **retrieval agent**, and a **Q&A agent**.
- **FR-33** **Extraction** (structuring raw input, identifying action items) and
  **writing** (choosing what to change in which section) are capabilities
  coordinated by the orchestrator, not separate agents.
- **FR-34** The orchestrator identifies the target project with the help of the
  **retrieval agent** (searching the knowledge base); there is no separate
  classifier agent.
- **FR-35** The roster may be extended with **additional specialized agents** (by
  domain or by kind of work), defined as part of the product; such additions are
  beyond the committed core.

### Autonomy and control

- **FR-21** The agents' autonomy is **configurable per tool**; a setting applies
  uniformly regardless of which agent invokes the tool (autonomy is **not** scoped
  per individual agent).
- **FR-22** **By default**, an agent's changes are proposed and require **user
  confirmation** before taking effect.
- **FR-23** The user can grant **full autonomy**, or **auto-approve specific
  tools** (for example, tools that only read data) while still confirming others.
  A **tool** is an MCP read/write operation over the knowledge base or the task
  platform.

### Conversation and interaction state

- **FR-46** Interaction happens in a **single assistant conversation** (not
  per-project threads); the platform **persists conversation history** as a **log**
  the user can revisit, which is **not** a knowledge source.
- **FR-47** Under the default propose-and-confirm autonomy (FR-22), unconfirmed
  proposed changes are held in a **durable review queue** the user can act on later,
  and in-flight clarifications **survive across sessions and reconnects**.

### Reliability and consistency

- **FR-52** The effects of an input (knowledge-base section writes and task changes)
  are applied **best-effort, per effect, with tracked status**; a success is durable
  and is **not** rolled back if a sibling effect fails. There is **no all-or-nothing
  guarantee** — across systems or within the knowledge base.
- **FR-53** **No silent failures:** any failed effect or failed input processing
  becomes a **retriable item** in the durable review queue (FR-47), and retained raw
  inputs (FR-49) allow the input to be **re-processed**.
- **FR-54** When the **task connector is unavailable**, knowledge-base effects still
  apply and task effects **wait as retriable items**, converging via reconciliation
  (FR-30).
- **FR-55** **Retries are reconciled** against the project↔task mapping and current
  state (FR-51) so a partially-succeeded effect is not duplicated.
- **FR-56** The knowledge base and the external task platform are **eventually
  consistent**; the project↔task mapping and reconciliation are the convergence
  mechanism, and inconsistencies are **surfaced** to the user, not silent.

### Non-functional

- **NFR-1** **Persistence.** Knowledge base information — and Personal Brain's
  other authoritative state, including the **project↔task mapping** — persists
  **permanently**: it is retained across sessions and indefinitely, until the user
  changes or deletes it.
- **NFR-2** **Transparency.** Everything an agent knows is visible to the user, a
  direct consequence of the single-source-of-truth principle (FR-6).
- **NFR-3** **Extensibility.** Task connectors are pluggable and custom sections
  are user-definable, without the user writing code.
- **NFR-4** **Model-agnostic.** The architecture supports using different
  underlying models for different agents (FR-20), avoiding single-model lock-in.
- **NFR-5** **Collaboration-ready (future).** The design should avoid choices that
  would preclude later multi-user sharing at the project or workspace level.
- **NFR-6** **Deployment.** Personal Brain is **cloud-hosted**: backend,
  databases, and any self-hosted models run on **managed PaaS** infrastructure;
  users access it through **thin web and mobile clients**, and agents execute
  **server-side**.
- **NFR-7** **Model sourcing.** The platform supports both **third-party hosted
  model APIs** and **self-hosted open-source models** on its own infrastructure,
  selectable **per agent** (reinforces FR-20 and NFR-4).
- **NFR-8** **Authentication.** Users authenticate via **email-based** sign-in or
  **OAuth** (social) sign-in; enterprise SSO is a future option.
- **NFR-9** **Data protection.** Data is encrypted **in transit (TLS)** and **at
  rest**; connector credentials and model API keys are held in a **managed secret
  store**, never in the knowledge base.
- **NFR-10** **Third-party model use.** Third-party model providers are used only
  under **no-training / zero-retention** terms. The baseline permits third-party
  APIs for all agents; **per-workspace/project privacy levels** that confine
  sensitive content to self-hosted models (NFR-7) are a reserved future extension.
- **NFR-11** **Interaction state.** **Knowledge** (in the knowledge base, FR-6) and
  **interaction state** (conversation transcript, pending proposals, in-flight
  clarifications) are distinct. Agents are **grounded in the knowledge base**;
  recent conversation is only immediate context, never long-term memory the
  knowledge base lacks. Conversation history persists as a **log**; pending
  proposals and clarifications persist **durably until resolved**.
- **NFR-12** **Reliability.** No input or effect is lost on failure: failures are
  **surfaced and retriable** (FR-53), retained raw inputs permit **re-processing**
  (FR-49), and the knowledge base and task platform **converge to consistency** via
  the mapping and reconciliation (FR-56).
- **NFR-13** **Scale.** The platform targets single-user scale — order **hundreds of
  projects** and **thousands of sections**. Retrieval assembles a working set across
  the referenced projects rather than loading whole projects (FR-61); **semantic
  search** across the knowledge base is a first-class mechanism at this scale.
- **NFR-14** **Performance.** Interactive operations respond **conversationally** (a
  first response within a few seconds); heavy captures are **asynchronous** with no
  interactive deadline; the platform favors **responsiveness and durability of work**
  over instant completion.
- **NFR-15** **Configuration scope.** Configuration (per-tool autonomy, connector,
  privacy/model-sourcing) is **account-global** in the current scope; **per-workspace
  scoping** is a reserved future extension.

## Open questions

_Things that are ambiguous or not yet decided. We track them here so they aren't
lost, and resolve them into the sections above as decisions are made._

- **Project status set.** Projects have a status (for example, preparing, active,
  closed). Whether this exact set is final or extensible, and the precise rules
  for how each status affects scope and agent behavior, are not yet pinned down.
- **Project templates (future).** Custom sections are defined **per individual
  project**. Reusable project templates — selectable when creating a new project —
  are a possible future direction, not part of the current scope.
- **Future additions to the roster.** The committed core is the orchestrator,
  retrieval, and Q&A agents, with extraction and writing coordinated by the
  orchestrator. Still open: which additional specialized agents (by domain or kind
  of work) the product will add, and the mechanics by which a new agent plugs into
  the orchestration (how it is selected and how its output is integrated).
- **Orchestration details.** The orchestrator model is settled (single entry
  point; resolve the project from explicit indication, inference, or by asking the
  user; select and invoke agents; agents return results to the orchestrator, which
  applies changes under the autonomy policy). Still to refine: how confidently the
  orchestrator must infer a project before acting rather than asking the user.
- **Task connector capability contract.** A connector must let agents create,
  update, and query tasks; map a Personal Brain project to the platform's native
  project or section; and, where available, deliver task-change events. The
  precise minimum contract — and how to handle platforms that lack a capability
  (for example, no event push) — is still to be specified. (Each platform is
  assumed to provide projects or sections to map onto.)
- **Number of task platforms.** The current scope assumes a **single connected
  task-management platform** for the user. Whether a user could connect more than
  one — for example, a different platform per workspace, or a per-project choice —
  is not yet decided. Supporting multiple would require the project↔task mapping
  and task routing to record which platform each task lives in.
- **Roles, collaboration, and sharing (future direction).** Multi-user
  collaboration — sharing a project's (or workspace's) information between users —
  is a planned future direction, not part of the current scope. The current
  product is effectively single-user. When collaboration is taken on, decisions
  will be needed about the unit of sharing, how users gain access, the set of
  roles and permissions (an **administrator** role has been mentioned for
  configuring connectors), and how concurrent contributions are handled. The
  current design should avoid choices that would preclude this later.
- **Planning capabilities (deferred).** Planning — organizing tasks across
  projects — remains a product aim, but its concrete capabilities (scheduling,
  prioritization, reminders, calendars, etc.) are intentionally deferred. The
  planning agent was introduced only as an example of an orchestrated specialized
  agent; taking it up later will require dedicated agent specialization.
- **Retention windows (deferred).** It is decided that raw inputs are retained as
  source artifacts (FR-49) and conversation history as a log (FR-46), both
  user-deletable. Still open: the default **retention windows / auto-purge policy**
  for these artifacts and the conversation log. Per-workspace/project **privacy
  levels** remain a reserved future extension, not yet designed.
