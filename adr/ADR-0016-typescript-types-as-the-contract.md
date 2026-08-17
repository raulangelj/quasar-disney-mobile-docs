# ADR-0016: TypeScript interfaces are the contract now; OpenAPI at backend engagement

**Status:** Accepted
**Date:** 2026-08-17

## Related documents

- `architecture/11-interface-contracts.md` (§2, §3, §11 — the contract this ADR governs)
- `architecture/03-architecture-overview.md` §5 (the API module is the only network-bound boundary)
- `architecture/04-data-model.md` §7 ("types in the API module *are* the schema")
- ADR-0002 — In-app API module with axios; no backend component
- ADR-0007 — Shared Container and Card types on both storefront GETs
- OQ-10 (backend team accepts the contract), OQ-18 (application repo name)

## Context

Session 1.11 found exactly one boundary in this system that will ever cross a network: the API
module's five operations (login, `/me`, HomeFeed, Continue Watching, and horizontal `resources`
paging). Everything else is an in-process import governed by doc 03 §8's import rules, a
library API, or a config file.

That boundary needs a specification, because three separate parties build against it: the mock
adapter that implements it today, the unit tests session 1.12 will specify, and — in Phase 3 — a
backend team that does not exist yet and has not been consulted (**OQ-10**).

The question is what form that specification takes. Three options were weighed:

**A. TypeScript interfaces plus prose tables.** The types in the API module are the schema, as
doc 04 §7 already asserts. Doc 11 documents paths, payloads, envelope, and error semantics as
Markdown tables. Zero build tooling.

**B. OpenAPI 3.1, design-first.** A `contracts/openapi.yaml` checked into the application repo is
the source of truth; the TypeScript types are written or generated to match.

**C. Hybrid.** TypeScript is the authoring source; an OpenAPI document is generated from it (via
Zod, `typescript-json-schema`, or similar) and checked in.

Three constraints bear on the choice. First, **the application repo does not exist yet** — OQ-18
has not named it, and `registries/repos.yml` lists only the two docs repos. Second, there is a
**fixed stakeholder sign-off on 2026-08-18**, and Phase 1a's scope is auth plus a storefront, not
tooling. Third — and this is the constraint that argues *against* the choice made below — this
project exists explicitly as a **methodology showcase and migration template** for the team's real
streaming product (doc 01), so what it demonstrates matters beyond what it delivers.

## Decision

**1. TypeScript interfaces in the API module are the authoring source of truth. Doc 11 is the
consumer-facing contract of record. No OpenAPI document is written in Phase 1.**

**Rationale.** The value of a machine-readable contract is realized by a *second implementer* who
generates code from it. There is no second implementer today, and there will not be one before the
sign-off date. An OpenAPI file written now would be consumed by nobody, verified by nothing, and
would sit in a repo for the months between Phase 1 and Phase 3 accumulating divergence from the
TypeScript that the running app actually uses — arriving at the backend team's desk as a document
that looks authoritative and is quietly wrong. The thing with genuine value *now* is locking the
JSON names, envelope, and error codes so the mocks and the tests cannot disagree, and doc 11's
tables do that at zero tooling cost.

**Alternatives.** *B (OpenAPI now)* buys a codegen-ready artifact for a consumer who is not yet
engaged, at the cost of authoring time before a fixed date and a hand-maintained YAML that can
diverge from the TypeScript with nothing checking it. *C (hybrid)* is the technically best
end-state and was the closest call: it eliminates the divergence risk that sinks B. It was
rejected on timing rather than merit — it adds a dependency and a generation step to a repository
that does not exist, days before a sign-off, and the generated artifact would still have no
consumer. C is the natural shape of decision 2 below.

**Reversibility.** Low cost to reverse. Going from typed interfaces to a generated schema is
mechanical; the field names, optionality, and enum values are already decided in doc 11 §7.

**2. Promotion to OpenAPI 3.1 is triggered by OQ-10 — the moment the backend team is engaged.**

At that point the OpenAPI document becomes the contract of record and doc 11 becomes the narrative
around it. This is recorded as item 1 on doc 11 §14's Phase-3 checklist alongside the seven other
decisions parked on the same trigger, so it is reached by walking a list rather than by someone
remembering this ADR exists.

**3. The strategy is conditional on typed fixtures, and that condition is binding.**

Mock fixtures are declared `Container[]` / `Card[]` — never `any`, never untyped JSON imports
(doc 11 §11.2). This is not a style preference. If fixtures are loosely imported, `tsc` checks
nothing about them, the types describe an intent no tool verifies, and "TypeScript is the
contract" becomes decorative. The entire decision above rests on this line holding.

## Consequences

**Easier.** No build tooling, no schema-generation step, and no second artifact to keep in sync
before 2026-08-18. The contract is enforced by the type checker the project already runs, on every
build, with no CI required — which matters because there is no CI until Bitrise (OQ-05). Adding a
field is an edit in one place.

**Harder.** The backend team, when engaged, receives Markdown tables and TypeScript rather than a
document they can generate a server stub from. Doc 11 §7 is written to be transcribable —
explicit types, explicit optionality, explicit enum members — but transcription is still manual
work, and the first thing OQ-10 should produce is decision 2's promotion.

**New risk.** The contract of record (docs hub) and the authoring source (an application repo that
does not exist) live in different repositories, and the normative one **changes hands** at the
scaffold STEP. Doc 11 §2.2 states the handover rule and §3.1 places an explicit obligation on the
scaffold STEP to create `src/api/types/` from doc 11's tables and link back. If that obligation is
skipped, the types get written from a memory of a conversation rather than from a specification,
and the two drift from their first commit.

**Acknowledged tension.** This project is a migration template, and "we wrote TypeScript
interfaces" is a weaker lesson to carry into the production streaming app than "we wrote a
contract." That argument is real and it is the strongest case for option B. It is outweighed here
by the judgment that shipping an unconsumed, unverified schema would teach a *worse* lesson —
that a contract is a file you produce rather than an agreement you keep — and by decision 2 making
the upgrade a scheduled event rather than an aspiration. If the showcase intent is later weighted
more heavily than the delivery date, this ADR should be superseded rather than amended.
