# ADR-0007: Shared Container and Card types on both storefront GETs

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/04-data-model.md`
- `architecture/05-scaling-performance.md`
- ADR-0005 (pagination hooks)
- ADR-0006 (two authenticated GETs, client composition)

## Context

STEP-1.4 modelled HomeFeed as `{ hero, carousels }` with `Carousel.items: Title[]`, and
Continue Watching as a list of Titles-plus-progress — not a HomeFeed variant. That split of
*endpoints* is still right (per-user CW vs catalog). The split of *types* is not: the UI is
one row component and one tile component, with chrome selected by variant (`hero`,
`progress`, portrait/landscape).

A later production catalog will look like “array of containers, each with cards,” not two
client-only envelopes. Naming `items` vs a distinct CW row type would duplicate components
and hide that shape.

## Decision

1. **HomeFeed** (JWT) returns a **`Container[]`**. The spotlight is a container with
   `variant: "hero"`. First page remains one hero + 15 other containers.
2. **Continue Watching** is still a **separate JWT GET** (ADR-0006). Its payload includes an
   attribute of the **same** `Container[]` type. Typically one container,
   `variant: "progress"` (timeline chrome).
3. A **Container** carries its own **name** and metadata, plus **`resources: Card[]`**
   (not `items`).
4. A **Card** carries the content **name** plus other content fields. Extra production
   fields are TBD (OQ-26); Phase 1 mocks fill the UI working set in doc 04.
5. Storefront **reuses** one Container component and one Card component. Variant selects
   chrome. Continue watching is **not** appended into the HomeFeed *response*.

**Rationale.** One type on the wire matches component reuse and keeps silent CW reload a
small replacement of the `progress` container. `resources` is the field name locked for
cards-in-a-container; JSON paths stay 1.11.

**Alternatives.** Keep `{ hero, carousels }` + a distinct CW item type — rejected; two UI
stacks. Put CW inside HomeFeed as another container — rejected; cannot silent-reload CW
without refetching catalog (ADR-0006). Invent Card fields now — rejected; unknown.

**Reversibility.** Low if mocks and components speak Container/Card from the first
storefront commit. High if Phase 1 ships Title/Carousel/`items` and Phase 3 renames
everything.

## Consequences

- Doc 04 v0.2.0 renames Title → Card, Carousel → Container, `items` → `resources`.
- ADR-0006 composition formula is amended to operate on `Container[]`.
- Session 1.11 names the JSON attribute that holds `Container[]` on each endpoint.
- Contract & mock STEP authors two fixtures of the same Container/Card shape.

## Amendment (2026-08-17 — STEP-1.11, Interface Contracts)

The original decision above stands unchanged. Session 1.11 resolved the three things it left to
1.11, recorded in `architecture/11-interface-contracts.md`:

1. **The JSON attribute holding `Container[]`** (Consequences, bullet 3) is **`data`**, inside the
   envelope `{ data, nextCursor }` — the same envelope on every paginated response, never a bare
   top-level array (doc 11 §6.1).
2. **`Card`'s content-name field is `title`, not `name`** (Decision 4). `Container.name` /
   `Card.title` keeps rows and content distinguishable when reading a payload. Doc 04 §1.2 is
   updated to match.
3. **OQ-26 is closed** (Decision 4): doc 04 §1.2's working set *is* the Card, with §1.3's three
   progress fields as optional members populated only inside a `progress` container
   (doc 11 §7.1). No speculative production fields were added.

One thing this ADR's Decision implies but never states: because a Container carries `resources`
that page independently, there is a **fifth operation** — `GET /containers/{id}/resources`
(doc 11 §5). Each Container therefore carries its **own** `nextCursor` alongside the envelope's
(doc 11 §6.2).
