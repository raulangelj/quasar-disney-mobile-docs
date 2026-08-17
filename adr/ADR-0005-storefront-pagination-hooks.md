# ADR-0005: Storefront pagination owned by feature hooks

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/15-native-app-architecture.md`
- `architecture/03-architecture-overview.md`
- ADR-0002 (API module + middleware is the only I/O path)
- Sessions 1.4 / 1.11 (wire format)

## Context

The storefront is a row-stack of carousels. Loading every row and every tile in one mock
payload would hitch on device and teach the wrong integration shape: a real catalog API
pages. Performance guardrails (virtualized lists) only work if the data layer also pages.

The team already forbids screens and hooks from importing axios. Pagination state still
needs a single, organized place or it will leak into list components.

## Decision

1. **The storefront paginates from day one** — vertically (pages of rows) and horizontally
   (pages of tiles in a carousel).
2. **Storefront-owned hooks** are the organized API for page state. A screen may call
   `useHomeFeed` / `useCarouselPage` (names illustrative). Each hook exposes
   `{ items, loadMore, hasMore, isLoading, error }` (or equivalent).
3. **`loadMore` dispatches to middleware**, which calls the API module. Mocks return
   paginated pages with latency and injectable failure — same as any other call (ADR-0002).
4. **Pagination cursors live in memory** with the content slice, not in secure storage.
5. **Cursor vs offset and envelope field names are not decided here.** Sessions 1.4 and
   1.11 lock the wire format (OQ-19). This ADR only requires that the contract *is*
   paginated.

**Rationale.** A hook keeps list components dumb and keeps the I/O path identical to login:
dispatch → middleware → API module. Paginated mocks mean Phase 3 does not grow a paging
layer that screens already assumed away.

**Alternatives.** One-shot home payload — rejected; unbounded image decode and a contract
that cannot page. Screen-level `onEndReached` calling the API module — rejected; breaks
the middleware boundary. A shared-kernel `usePagination` as the only API — optional later
as a helper; the *owner* of feed/carousel page state stays the storefront feature.

**Reversibility.** Low if mocks and hooks speak pages from the first storefront commit.
High if Phase 1a ships a full-catalog fixture and must be retrofitted before Phase 3.

## Consequences

- Contract & mock STEP (Dev B) must page fixtures, not return the whole catalog.
- Session 1.4 models page tokens / offsets; 1.11 puts them on the envelope.
- Virtualized carousel lists call `loadMore` on end-reached; they do not fetch.
- Tests cover reducer/middleware/hook paging (Phase 1a test gate), not UI virtualization.
