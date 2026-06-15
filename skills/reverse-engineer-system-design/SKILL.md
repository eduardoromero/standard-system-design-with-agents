---
name: reverse-engineer-system-design
description: Reverse-engineer an existing codebase into a Systems Design Specification (or audit/update an existing spec for divergence from code). Use when asked to "document how this system works", "reverse engineer this repo into the design spec", "audit the spec against the code", or "keep the spec in sync". Produces a code-faithful, fully-cited spec following the Systems Design Specification Standard.
---

# Reverse-Engineer a System into the Design Spec

You turn real code into a true spec — and keep it true. The output follows
[`systems-design-specification-standard.md`](../../systems-design-specification-standard.md).
**Read §0 (Evidence & Authoring Discipline) of the standard before writing anything.** Your one job is to be *correct*; a pretty spec with one fabricated number is a defect.

## The cardinal rule
**Every claim is traced to a source (`file.ts:line`, a config key, a migration) or it is not written.** When unsure, open the file and read it. "I could not find X" is a valid spec entry; a confident guess is not.

---

## Workflow A: Reverse-engineer from scratch

### Step 1 — Orient (Phase 0 first, always)
- Read the README, package manifests, and top-level dirs. Identify **where the real implementation lives vs. thin re-export/facade packages** (a published package that just `export *`s from elsewhere is not the implementation — follow it).
- Write the **System Thesis**: one or two sentences naming the single architectural idea that explains most of the complexity. If you can't, keep reading — you don't understand it yet.
- List the 5–10 files a newcomer must read first.

### Step 2 — Read the spine (don't fan out blindly)
Read, in full or in targeted slices: the server/app **entry point**, the **schema/DDL** (`db.ts`, migrations, `CREATE TABLE`), the **route table(s)**, the **auth/policy** module, and any **core domain doc** (a provenance/protocol/ADR file). These five give you ~80% of the spec.

### Step 3 — Harvest with grep, then verify by reading
- Enumerate the **full route surface**: `grep` every `.get(|.post(|.put(|.delete(` (or framework equivalent). Don't stop at README-highlighted routes.
- Enumerate **event types, env vars, limits**: grep for emitted event strings, `process.env.*`, and numeric defaults (`|| 3000`, `RATE_LIMIT`, `LIMIT`, `TIMEOUT`, `MAX`).
- For each grep hit you intend to document, **open the handler/definition and confirm its actual behavior** (return shape, status codes, side effects). Grep locates; reading confirms.

### Step 4 — Fill the standard, phase by phase
Phase 0 → 5 in order. As you write:
- **Numbers:** label every frequency/latency as `measured` / `configured (cite)` / `intent — inferred`. Never write a naked SLA. If a system ships no telemetry, say so.
- **Schema:** copy exact types/constraints/**all** enum values from the DDL. Verify deletion behavior (`CASCADE` vs soft-delete vs `RESTRICT`) — read it, don't assume.
- **Contracts:** read the handler before stating what an endpoint returns (JSON block list ≠ rendered HTML). Note fields deliberately omitted from DTOs (secrets, hashes, blobs, health flags).
- **Auth:** document the model (§4.0) before the routes; capture per-route asymmetries (propose vs approve).
- **Diagrams:** include valid Mermaid for C4 context/container/component and the ER relationships.
- **Domain invariants:** state the business rules and cite where they're enforced.

### Step 5 — Self-check and ship
Run Appendix B of the standard. Do not ship until every box is ✅, especially "no claim without a citation; no fabricated numbers; unknowns marked."

---

## Workflow B: Zero-divergence audit (spec already exists)

Use when asked to check a spec against the code or update it after changes. Identify divergence in this order:

1. **Schema drift** — grep `CREATE TABLE` / migrations; diff each table/column/constraint/enum against Phase 2.1. Flag missing columns, wrong defaults, undocumented tables, wrong/missing enum values, and **wrong deletion behavior** (a common spec lie: claiming `CASCADE` where the code soft-deletes).
2. **Route drift** — grep all route registrations; verify every spec route exists and every code route is documented. Flag deprecated aliases, renamed params, and undocumented owner/admin routes.
3. **Limit/timeout drift** — find env parsers and defaults; cross-check Phase 5.1. Flag any number where spec ≠ code (and any number in the spec with no source).
4. **Concurrency/contract drift** — check locks/transactions/idempotency and error status codes against Phase 2.4 and 5.2.

**Resolution:** locate the source-of-truth line; make a **surgical edit** (preserve unaffected diagrams/indexes/prose); re-cite the new line; report only the corrected gaps with clickable links.

---

## Anti-patterns to refuse (these are how specs become lies)
- ❌ Inventing throughput/latency to fill the matrix. → Label `UNKNOWN — needs telemetry`.
- ❌ Describing an endpoint's response from its name. → Read the handler.
- ❌ Assuming `CASCADE`/enum values/optionality. → Read the DDL.
- ❌ Documenting only the marketed routes. → Grep the whole surface.
- ❌ Dumping DB rows as the API DTO. → DTOs are contracts; note omissions.
- ❌ Copying claims from another spec without re-verifying against this codebase.

## Output
A single Markdown spec following the standard, saved to the repo (e.g. `docs/SYSTEM-DESIGN-SPEC.md`), with Mermaid diagrams, a Glossary, and the Self-Check appendix. Tell the user where Mermaid will/won't render, and offer to link it from the README.
