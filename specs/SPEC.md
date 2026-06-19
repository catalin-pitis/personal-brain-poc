# Personal Brain — Specification

> Living specification for the Personal Brain platform. This document evolves as
> requirements are discovered and refined. See [`CLAUDE.md`](../CLAUDE.md) for how
> we work on it.

## Overview

Personal Brain helps a user centralize and structure information about their
projects and activities, and organize and plan that work.

Its defining characteristic is that it is **agent-mediated**. Rather than the
user manually structuring information, the user communicates in natural language —
typed or spoken — and by sharing documents, and an agent maintains a structured
knowledge base on their behalf. The intent is to shift the user's time away from
organizing information and toward thinking and making decisions based on a
knowledge base that is always current.

Two jobs sit at the core of the product:

1. **Knowledge** — capturing and structuring information about projects into a
   knowledge base.
2. **Planning** — organizing activities (tasks) and planning the work.

These are linked: the planning is about the tasks that advance the same projects
the knowledge base describes.

## Core concepts

_The shared vocabulary of the product. Definitions are filled in as they are
decided; terms still under discussion are marked as open._

- **Knowledge base** — the persistent, structured store of information about the
  user's projects. It is the **single source of truth**: agents read from and
  write to it, and keep no private state of their own, so everything an agent
  "knows" is visible to the user. It is maintained primarily by the agents but is
  directly readable by the user. (Sharing project information between multiple
  users is a planned future direction, not part of the current scope.)
- **Agent** — an actor that interprets what the user communicates, decides what
  it pertains to, and updates the knowledge base accordingly. The platform
  provides **multiple specialized agents** rather than one; agents are the
  primary means of getting information *into* and *organized within* the
  knowledge base.
- **Orchestration** — the platform's coordination of its specialized agents:
  routing a given input to the agent(s) best suited to handle it, and combining
  their work when an input spans several concerns or projects.
- **Workspace** — a grouping of projects, and the top-level container in the
  knowledge base. A workspace may hold information common to all the projects
  within it (for example, contact persons and their roles), so shared context is
  not repeated in each project.
- **Project** — a larger, longer-lived effort the user works toward, and the
  primary unit around which information is organized. Every project belongs to a
  workspace.
- **Activity** — a task: a discrete unit of work the user does. Every task
  belongs to a project. Tasks are **not** stored in the knowledge base; they are
  managed in an external task-management platform that Personal Brain integrates
  with. Activities are what the planning side of the product organizes.
- **Task-management platform** — the external system of record for tasks.
  Personal Brain does not store tasks itself; it reads and updates them through a
  configurable **connector**. The specific platform is not fixed: the user (or,
  in a future multi-user setting, an administrator) configures a connector to
  whatever task-management platform they use, so the integration is pluggable (an
  MCP connector is the intended mechanism). Agents act on tasks there on the
  user's behalf.

## Workspaces and projects

Projects are grouped into **workspaces**. A workspace is the top-level container
in the knowledge base, and every project belongs to one. The hierarchy is
therefore *workspace → project → project information*.

A workspace may also hold information **common to all its projects** — for
example, the contact persons involved and their roles — so that shared context
does not have to be repeated in each project. Within a workspace, each project
then carries its own information, structured as described below.

## Project information

Information about a project is organized into **named sections**. A project
starts from a **minimal built-in structure** and can be **customized**: the user
can add sections to track whatever matters for that project. The structure is not
fixed by the platform beyond a small default.

The minimal built-in sections are:

- **Open topics** — things currently open or unresolved on the project.
- **Decisions** — a record of decisions made, kept for later reference.
- **Project log** — a chronological history of what has happened on the project,
  for reference.

Beyond these, the user can add their own sections — for example status
information, KPIs, or anything else worth tracking about the project.

The agents maintain these sections as information comes in. Because the structure
(including any custom sections) is part of the knowledge base, it is directly
readable by the user.

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

Agents are **defined as part of the product** during development. Normal users
do not define, configure, or compose agents; they interact with the platform and
the orchestration selects the right agents on their behalf.

The knowledge base holds **project information** and is the single source of
truth for it. **Tasks live outside** the knowledge base, in an external
task-management platform that the product integrates with through tools (via
MCP). Agents act on tasks there rather than storing them locally.

A dedicated **planning agent** helps the user organize their tasks **across
projects**, working **through a conversation** with the user. It operates over
the tasks in the external platform — which may span several projects — and helps
the user decide how to organize them interactively, rather than applying a fixed
algorithm. The finer capabilities it offers (e.g. prioritization, scheduling,
surfacing what is next or overdue) are still to be defined.

Capture crosses the knowledge/task boundary in **both directions**. A single
input — for example shared meeting notes — may **both** update project
information **and** create or update tasks in the external platform: conclusions
update the knowledge base while action items become tasks. Conversely, when a
task changes or is completed in the external platform, that change is **reflected
back** into the knowledge base (for example as a Project log entry, or by
resolving an Open topic).

How much the agents act on their own is **configurable**. By default, an agent
**proposes** its changes and the user must **confirm** them before they take
effect. The user can choose to grant more autonomy — allowing the agent to apply
changes automatically, either fully or for **specific tools only**. For example,
the user might auto-approve tools that merely *read* data while still confirming
tools that *write* (create or modify knowledge or tasks).

The user can communicate with the agents by **typing or by speaking**. Spoken
input is supported as a first-class way to interact, because free-form speech is
often a faster and more natural way to capture what happened than writing it out.

Importantly, agent mediation applies to *maintaining* the knowledge base, not to
*accessing* it: the user is always able to read the project information directly,
without an agent in the loop.

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

4. **Contextual retrieval.** When preparing for a meeting or a new activity, the
   user asks the agent for context — for example, the status of a project or its
   open points — and the agent provides the requested details.

5. **Direct reading.** The user reads the project information directly, without
   the agent in the loop.

6. **Task planning.** Through a conversation with the planning agent, the user
   organizes their tasks across projects — reviewing and arranging the work that
   lives in the external task-management platform.

## Requirements

> A first pass derived from the decisions above. Requirements are stated only
> where decided; where an open question blocks a detail, it is noted inline.
> The `FR-`/`NFR-` identifiers are stable handles for reference as the spec
> evolves.

### Knowledge base and structure

- **FR-1** The platform organizes knowledge as a hierarchy of **workspaces →
  projects → per-project named sections**.
- **FR-2** Every project belongs to a workspace. _(Whether a project may belong
  to more than one workspace is open.)_
- **FR-3** Each project starts from a minimal built-in structure consisting of
  the sections **Open topics**, **Decisions**, and **Project log**.
- **FR-4** The user can add **custom sections** to a project to track additional
  information (for example, status or KPIs).
- **FR-5** A workspace can hold information **common to all its projects** (for
  example, contact persons and their roles).
- **FR-6** The knowledge base is the **single source of truth** for project and
  workspace information; agents retain no private state outside it.
- **FR-7** The user can **read** project and workspace information directly,
  without an agent in the loop.

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
  or resolution of an Open topic). _(Sync mechanism is open.)_

### Retrieval

- **FR-13** The user can ask an agent for **contextual information** about a
  project (for example, its status or open points) and receive the requested
  details.

### Tasks and planning

- **FR-14** Tasks are stored in an **external task-management platform**, not in
  the knowledge base.
- **FR-15** Every task is **associated with a project**. _(How the association is
  represented and kept in sync is open.)_
- **FR-16** The platform integrates with the task-management platform through a
  **configurable, pluggable connector** (MCP is the intended mechanism).
  _(Minimum connector capabilities are open.)_
- **FR-17** A **planning agent** helps the user organize tasks **across projects**
  through conversation. _(Its finer capabilities — prioritization, scheduling,
  surfacing what is next or overdue — are open.)_

### Agents and orchestration

- **FR-18** The platform provides **multiple specialized agents**, defined as part
  of the product. End users do not define, configure, or compose agents.
- **FR-19** The platform **orchestrates** its agents — selecting the appropriate
  agent(s) for an input and combining their outputs. _(Routing logic is open.)_
- **FR-20** Different agents may be powered by **different underlying models**.

### Autonomy and control

- **FR-21** The agents' autonomy is **configurable**.
- **FR-22** **By default**, an agent's changes are proposed and require **user
  confirmation** before taking effect.
- **FR-23** The user can grant **full autonomy**, or **auto-approve specific
  tools** (for example, tools that only read data) while still confirming others.

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

- **Workspace details.** Projects are grouped into workspaces, which can hold
  common information. Still to decide: whether a project can belong to more than
  one workspace, what kinds of common information a workspace holds (contacts and
  roles are one example), and how workspaces are created and managed.
- **Scope of project customization.** Custom sections are supported. It is not
  yet decided whether they are defined per individual project or as reusable
  templates shared across projects, and who may add or change them (relates to
  roles below).
- **What the agent specializations are.** It is decided that there are multiple
  specialized agents, but which specializations exist (e.g. by task type such as
  capture vs. retrieval, by domain, by project) is not yet defined.
- **How orchestration routes work.** How the platform decides which agent(s)
  handle a given input — and how their outputs are combined when several are
  involved — is not yet specified.
- **What a task connector must support.** The task integration is a pluggable
  connector. The minimum capabilities a connector must provide (e.g. create,
  update, query tasks) and how differences between platforms are handled are not
  yet specified.
- **How projects and tasks link.** Projects live in the knowledge base; tasks
  live externally. How a task is associated with a project, and how that
  association is created and kept in sync, is not yet specified.
- **Voice output.** Spoken input is supported. It is not yet decided whether the
  agents also respond by voice, or whether voice is input-only.
- **Roles, collaboration, and sharing (future direction).** Multi-user
  collaboration — sharing a project's (or workspace's) information between users —
  is a planned future direction, not part of the current scope. The current
  product is effectively single-user. When collaboration is taken on, decisions
  will be needed about the unit of sharing, how users gain access, the set of
  roles and permissions (an **administrator** role has been mentioned for
  configuring connectors), and how concurrent contributions are handled. The
  current design should avoid choices that would preclude this later.
- **Planning capabilities.** What "organize activities and plan" concretely
  means (scheduling, prioritization, reminders, calendars, etc.) is not yet
  specified.
