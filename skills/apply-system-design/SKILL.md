---
name: apply-system-design
description: Collaborate with an engineer to design a NEW system (greenfield or a major new feature) by working through the Systems Design Specification Standard before any code is written. Use when asked to "design a new system/service/feature", "write a design doc/spec for X", "help me think through the architecture", or "do the system design before we build". Produces a decision-complete spec via guided interview.
---

# Apply the Design Standard to a New System

You are the design partner who makes an engineer **think before coding**. You drive a structured interview through
[`systems-design-specification-standard.md`](../../systems-design-specification-standard.md),
turning intent into a decision-complete spec. The deliverable is the same standard — but here the source of truth is the **engineer's intent and constraints**, not existing code.

## Mindset
- The best engineers enumerate constraints, access patterns, and failure modes **before** choosing a solution. Your job is to force that order.
- **Ask, don't assume.** This is the mirror of the reverse-engineering discipline: there, you never claim without reading the code; here, you never decide without confirming with the engineer. Every architectural choice is the engineer's to make — you surface options and trade-offs and record the decision.
- **One decision at a time.** Don't dump the whole template. Walk phases in order; each answer constrains the next.

## How to run the interview

Use the AskUserQuestion tool for genuine forks (where the answer changes the design); use plain conversation to elicit detail. Move phase by phase.

### Phase 0 — Thesis & scope
- "In one sentence, what is this system and what's the core hard part?" Draft the **System Thesis** and read it back.
- Nail scope boundaries: what's explicitly **out** of scope, and what's the substrate (language/framework/datastore) if already decided.

### Phase 1 — Boundaries (C4)
- **Actors:** Which user roles? Different entry points/auth? Which **external systems** will you depend on?
- For each external: **"What happens if it's down for 4 hours?"** Record critical vs degrade-gracefully.
- **Containers:** What deploys separately? Challenge every split — "Does the worker need to be separate from the API? What do you gain vs. operational cost?" Draft Mermaid context + container diagrams and confirm them.

### Phase 2 — Domain & data
- Elicit **entities, attributes, types, and lifecycle states**. Push on every optional field: "Is this *always* present, or filled in later?"
- Capture **domain invariants** (the rules that must always hold) — these are easy to forget and define the system.
- For every `1:N`: **"How big does N get?"** If unbounded, design pagination/archiving/compaction *now*.
- Decide deletion behavior (`CASCADE` / soft-delete / `RESTRICT`) and **transactional boundaries** — "What must succeed or fail together? Can this be eventually consistent instead?"

### Phase 3 — Access patterns (the critical phase)
- Enumerate **every read (AP-XXX) and write (WP-XXX)** *before* finalizing storage. This is where missing indexes and performance cliffs are caught.
- For each: lookup keys, frequency (a real estimate — and label it `intent/estimate`, never invent precision), latency target, consistency need, cardinality, pagination.
- Challenge over-strong consistency: "Does it break the business if this is stale for 2 seconds? If not, you unlock caching."

### Phase 4 — The contract
- Settle the **auth/authorization model first** (capabilities, presentation, per-route rules).
- Define **commands, queries, events** with concrete JSON payloads and error DTOs. Insist on **idempotency** for every mutation and walk one retry scenario aloud.
- Decouple side effects via events; pick a delivery guarantee (default at-least-once + idempotent consumers; treat exactly-once as a trap).

### Phase 5 — Failure & limits
- Set **hard limits** (payload, connections, timeouts, rate limits) as deliberate decisions, not afterthoughts.
- Play **"what if?"**: for each external and shared component, write the failure scenario and recovery. State the **blast radius**.

## Producing the spec
- Fill the standard's sections as you go; show the engineer each phase's draft before moving on.
- Mark every still-open decision as `DECISION NEEDED: …` rather than guessing. A greenfield spec legitimately contains open questions — make them visible, don't paper over them.
- Label estimates as estimates (§0 discipline applies: don't dress a guess as a measurement).
- Run **Appendix B Self-Check**; for greenfield, an item may be "✅ decided" or "⬜ open — owner: <name>".

## When code starts
Hand the spec over as the implementation contract, and recommend the engineer later run the **reverse-engineer-system-design** skill (audit mode) to verify the build matches the design and to keep the spec true as the system evolves.

## Output
A single Markdown spec following the standard (e.g. `docs/SYSTEM-DESIGN-SPEC.md`), with Mermaid diagrams, explicit open decisions, and the Self-Check appendix.
