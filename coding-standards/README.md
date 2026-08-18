# Coding Standards

Per-language engineering standards for quasar-disney-mobile: naming, layout, error handling,
logging, documentation, and testing conventions. Code substeps should reference the relevant
standard so the codebase stays consistent regardless of who (or which agent) writes it.

## How this works
- **This directory has been reconciled to this project** in STEP-1.12. Doc 03's stack is
  **TypeScript, only**, so the seven unused language defaults that shipped with Throughstone
  (`python.md`, `go.md`, `rust.md`, `dart.md`, `java.md`, `csharp.md`) and cross-cutting `sql.md`
  were **pruned** — doc 04 §3 records that there is no relational database, no SQLite/MMKV, and no
  server datastore. What remains is the three files in the table below.
- The remaining files are **no longer generic starting points** — each has been amended to match
  decisions already recorded in `architecture/` and `adr/`. `typescript.md` and `api.md` in
  particular reverse several shipped defaults. The reasoning is in
  [`../architecture/12-test-strategy.md`](../architecture/12-test-strategy.md) §10; read that before
  changing a rule back.
- **Where a rule here and an `architecture/` doc disagree, the architecture doc wins** and this file
  has drifted. `api.md` is subordinate to `architecture/11-interface-contracts.md`, which is the
  contract of record.
-   If a standard reflects a decision, link the ADR that records why — several rules here link
  ADR-0002, ADR-0003, ADR-0010, ADR-0016, ADR-0017, ADR-0018, ADR-0019, and ADR-0020.

## Documentation & comments  (all languages)
A project-wide rule; each language file shows the idiomatic *form* and the lint that
enforces it.
- **Docstrings are required on every class, method, and function** — and on fields/properties
  where the language documents them (e.g. Java fields, C# properties) — public *and* private.
  Say what it does, its parameters and return, and anything non-obvious about its contract.
  Describe what the code **actually does**, not what you set out to write — a docstring that
  drifts from the behavior misleads worse than silence. This is a gate: code without them
  isn't done.
- **Comment for the next reader.** As a guideline, expect roughly a comment every ~10 lines
  of non-trivial code — a readability *suggestion*, not a counted requirement. Explain the
  *why*; don't restate what the code already says. Cut narration (`// increment i`): it's
  noise, and it goes stale.
- **Keep comments and docstrings true** as the code changes — a stale one is worse than none.

## Files
Reconciled to this project's stack in STEP-1.12 (doc 12 §10).

| Language / area | File | Status | Applies because |
|-----------------|------|--------|-----------------|
| TypeScript | [`typescript.md`](typescript.md) | **Active — project-specific** | The one implementation language (doc 03) |
| Shell / Bash (cross-cutting) | [`shell.md`](shell.md) | **Active — lightly amended** | The hub ships `scripts/*.sh` and root `doctor.sh`; doc 09 §7.2 contemplates a `.env` completeness script |
| API design (cross-cutting) | [`api.md`](api.md) | **Active — substantially amended**; subordinate to `architecture/11-interface-contracts.md` | Doc 11's contract is REST-shaped and is delivered to a backend team (criterion A4); boundary B-B becomes real HTTP at Phase 3 |

**Pruned in STEP-1.12** — not used at any phase in the roadmap: `python.md`, `go.md`, `rust.md`,
`dart.md`, `java.md`, `csharp.md`, and `sql.md` (no relational database — doc 04 §3). If a later
phase introduces one of these languages, recreate the file from an existing standard's structure
rather than reaching for the deleted default.
