# Personal Brain

## What this is

Personal Brain is a platform that lets users maintain information about their
projects, manage their time and activities, and work collaboratively over a
shared knowledge base.

This repository is currently a proof of concept. The immediate focus is **not**
code — it is defining the product through written specifications. Implementation
will follow once the requirements are clear.

## How we work right now: specifications

All product requirements live as Markdown under [`specs/`](specs/).

We maintain a **single evolving specification**: [`specs/SPEC.md`](specs/SPEC.md).
It grows over time as requirements are discovered and refined. We do not split it
into many files yet; if it becomes unwieldy we will reorganize deliberately, not
preemptively.

There is **no fixed template**. The spec is prose-first. The priorities, in order:

1. **Clarity** — anyone reading a section should understand what the product does
   and why, without needing prior context from the conversation.
2. **Consistency** — terminology, structure, and level of detail stay uniform
   across the document. Define a term once and use it the same way everywhere.
3. **Traceability of decisions** — when a requirement reflects a deliberate
   choice (especially one that closed off an alternative), capture the reasoning,
   not just the conclusion.

### Working on the spec

- **Capture, don't invent.** The spec records what the user decides. When the
  user is thinking out loud, help shape and sharpen the idea, but don't silently
  promote a suggestion into a committed requirement. Distinguish what's decided
  from what's still open.
- **Surface open questions.** When something is ambiguous or underspecified, ask
  rather than guess, or record it explicitly as an open question in the spec.
- **Keep it current.** When a decision supersedes an earlier one, update the
  affected sections so the document never contradicts itself. The spec is the
  source of truth, not an append-only log.
- **Right altitude.** This is a requirements document — describe behavior,
  capabilities, and constraints. Avoid premature implementation detail (specific
  frameworks, schemas, APIs) unless a requirement genuinely depends on it.
- **Use the product's vocabulary.** As terms emerge (the core domain concepts,
  user roles, key objects), use them consistently and keep them defined.

## Conventions

- Specs are Markdown, written in clear English prose.
- Prefer editing the existing spec over creating parallel documents.
- When the user asks to "add a requirement," "spec out X," or similar, the work
  goes into `specs/SPEC.md`.
