# ADR-0006: Two authenticated storefront GETs composed on the client

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/04-data-model.md`
- `architecture/03-architecture-overview.md`
- `architecture/05-scaling-performance.md`
- ADR-0002 (API module + middleware is the only I/O path)
- ADR-0005 (pagination hooks; each endpoint still pages)
- ADR-0007 (shared Container / Card types)

## Context

A Disney+-style home is not one catalog. Continue watching is per-user and changes often;
the rest of the row stack (hero + standard rows) is catalog. Putting both in a single mock
payload would make a silent “did continue watching change?” refresh reload hero and every
other row, and would hide the split a real backend will expose.

The composed order is also a product rule: continue watching sits **directly under the hero**,
pushing the other rows down one — it is not “whatever the catalog API sent first.”

## Decision

1. **Two JWT-authenticated GETs** (paths locked in 1.11):
   - **HomeFeed** — first page is **one hero carousel + 15 other carousels**.
   - **ContinueWatching** — Titles with progress fields (`progress`, `remainingMinutes`,
     `episodeLine`). The request carries the JWT; the mock keys fixtures off that session.
2. **Storefront composes the vertical list:**
   `[HomeFeed.hero] + [ContinueWatching row] + [HomeFeed.carousels…]`.
   Continue watching is **not** a HomeFeed variant.
3. **Cold start / first paint after login:** the **app shell** shows a loading screen until
   `/me` + HomeFeed + ContinueWatching all complete, then paints. Avoids inserting the CW
   row after first paint (the 15 rows would jump down).
4. **Later visits to the storefront screen:** **silent refetch of ContinueWatching only**
   (stale-while-revalidate on that row). Hero and the 15 do not reload for that reason.
5. Each endpoint still **pages** (ADR-0005). Phase 1 HomeFeed mocks may send `nextCursor: null`
   after the 16-row first page.

**Rationale.** The client merge is cheap and matches production. Silent CW reload stays a
small call. Blocking first paint on both catalog calls is the right tradeoff for a
visual-fidelity POC with artificial mock latency.

**Alternatives.** One home payload including CW — rejected; cannot silent-reload CW without
refetching the catalog, and the backend will not look like that. Paint HomeFeed first and
splice CW in — rejected; layout jump under hero. Polling CW while the screen is visible —
rejected; refetch on appear is enough for the demo.

**Reversibility.** Low if mocks and hooks are two calls from day one. High if Phase 1 ships a
single fixture list and Phase 3 has to split it.

## Consequences

- Contract & mock STEP authors **two** feed fixtures and JWT-gated CW data.
- Session 1.11 names paths and envelope fields.
- Doc 02’s hero *UI* cost still stands (OQ-24); the **feed contract and composition** include
  hero from Phase 1.
- Shell owns the boot loader; Storefront owns silent CW reload. Neither is the no-internet
  overlay (ADR-0004).

## Amendment (2026-08-17 — STEP-1.5)

Decision **1–5 still hold** (two JWT GETs, client composition under hero, boot loader, silent
CW reload, paging). The **envelope** changes:

- HomeFeed is a **`Container[]`**, not `{ hero, carousels }`. Hero is a container with
  `variant: "hero"`.
- Continue Watching is a **`Container[]`** (typically one container),
  `variant: "progress"`. It remains a **separate API** — still **not** a member of the
  HomeFeed *response*.
- Composed list:
  `[ HomeFeed hero container ] + [ CW progress container ] + [ remaining HomeFeed containers ]`.
- Cards live in **`resources`**, not `items`. Shared types: **ADR-0007**.

The sentence in decision 2, *“Continue watching is not a HomeFeed variant,”* is amended to:
not a HomeFeed **payload** member. It **is** a Container variant (`progress`) for UI reuse.

## Amendment (2026-08-17 — STEP-1.14)

**OQ-24 is closed.** Phase 1a renders the hero container as a **3:4 portrait stand-in**. Full
spotlight chrome (peeking neighbors, title art, CTA) is Phase 2. Feed contract and composition
still include `variant: "hero"` from Phase 1.
