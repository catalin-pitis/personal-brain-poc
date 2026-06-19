# Personal Brain — Specification

> Living specification for the Personal Brain platform. This document evolves as
> requirements are discovered and refined. See [`CLAUDE.md`](../CLAUDE.md) for how
> we work on it.

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
  user's projects. It is the **single source of truth**: agents read from it and
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
does not have to be repeated in each project. Unlike a project, this
workspace-level common information does **not follow a predefined structure**; it
holds whatever shared context is relevant, in a flexible form. Within a workspace,
each project then carries its own information, structured as described below.

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

The agents maintain these sections as information comes in. Because the structure
(including any custom sections) is part of the knowledge base, the user can read
**and edit** it directly as well, by hand.

Each project also has a **status** — for example *preparing*, *active*, or
*closed*. Status affects scope: by default, **normal requests exclude closed
projects** (for instance, a cross-project status query or the routing of new input
considers only active and preparing projects), while the user can still include
closed projects explicitly when needed.

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
information directly, by hand, without an agent in the loop. Agent-mediated
maintenance and direct editing operate on the same single source of truth, so
hand edits are immediately visible to the agents (which keep no private state).

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
- **FR-37** Workspace-level common information has **no predefined structure**; it
  holds shared context in a flexible form (it does not use the project section
  model).
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

### Capture

- **FR-8** The user can provide information by **typing**, by **speaking**, or by
  **sharing documents** (for example, notes or meeting transcripts).
- **FR-9** The platform **interprets** user input and updates the relevant
  information without requiring the user to structure it manually.
- **FR-10** When an input spans **several projects**, the platform routes it to
  each relevant project and updates each accordingly.
- **FR-11** A single input can produce **multiple effects** — updating knowledge
  and creating or updating tasks — as warranted (for example, action items become
  tasks while conclusions update the knowledge base).
- **FR-12** Changes to a task in the external platform (for example, completion)
  are **reflected back** into the knowledge base (for example, a Project log entry
  or resolution of an Open topic). See FR-30 for timing.
- **FR-36** Voice is supported for **input only**; agents respond in text, not by
  voice.

### Retrieval

- **FR-13** The user can ask an agent for **contextual information** about a
  project (for example, its status or open points) and receive the requested
  details.
- **FR-40** By default, **normal requests exclude closed projects** from their
  scope; the user can include them explicitly when needed.

### Tasks and planning

- **FR-14** Tasks are stored in an **external task-management platform**, not in
  the knowledge base.
- **FR-15** Every task is **associated with a project**. Personal Brain maintains
  this association in its own internal **project↔task mapping**, independent of
  the external platform's organizational features. See FR-30 for sync timing.
- **FR-16** The platform integrates with the task-management platform through a
  **configurable, pluggable connector** (MCP is the intended mechanism).
  _(Minimum connector capabilities are open.)_
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

### Non-functional

- **NFR-1** **Persistence.** Knowledge base information persists across sessions.
- **NFR-2** **Transparency.** Everything an agent knows is visible to the user, a
  direct consequence of the single-source-of-truth principle (FR-6).
- **NFR-3** **Extensibility.** Task connectors are pluggable and custom sections
  are user-definable, without the user writing code.
- **NFR-4** **Model-agnostic.** The architecture supports using different
  underlying models for different agents (FR-20), avoiding single-model lock-in.
- **NFR-5** **Collaboration-ready (future).** The design should avoid choices that
  would preclude later multi-user sharing at the project or workspace level.

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
