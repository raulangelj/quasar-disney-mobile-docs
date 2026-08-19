# ADR-0004: Online-only client with a shell-level connectivity gate

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/15-native-app-architecture.md`
- `architecture/03-architecture-overview.md`
- `architecture/02-phasing-roadmap.md`
- `inputs/ui/streaming-reference-screens.md` (§7)
- ADR-0003 (auth persist is not an offline cache)

## Context

A native app is often assumed to need an offline store. This project is a mock-backed
stakeholder demo; downloads are Phase 4+. The Disney+ reference also includes a distinct
no-internet full screen (copy + `REINTENTAR`, auto-reload when connectivity returns). Those
are different problems: caching content vs. telling the user the network is gone.

If each feature drew its own offline UI, Auth and Storefront would duplicate a shell concern
and the restore path (auth vs storefront) would fork.

## Decision

1. **The app is online-only.** No content cache, no sync, no conflict resolution in Phase 1.
2. **Reachability is a shell concern.** `@react-native-community/netinfo` drives a single
   full-screen overlay owned by the app shell, matching the reference screen.
3. **Restore is by auth state:** on reconnect (or Retry), hide the overlay and show
   storefront if a session exists, auth otherwise. Refetch the active surface. The overlay
   sits on top of the current navigator; it does not unmount it.
4. **No-network is not a fetch error.** Feature error states (inline login error, storefront
   load failure) stay in their features. The gate is only for “the device has no network.”

**Rationale.** Session restore already lives in the shell (rehydrate → choose navigator).
Connectivity is the same class of decision. Online-only matches deferred downloads and an
in-process mock that is not a stand-in for disk.

**Alternatives.** Last-known-home cache — rejected; it is a second source of truth for
content and is unused on a demo that always has mocks. Per-feature offline screens —
rejected; duplicates the reference UI and the restore rule. Treating NetInfo as a failed
axios call — rejected; that would show the credentials inline error (or a spinner) instead
of the reference gate.

**Reversibility.** Low for the overlay ownership. Medium to add a later read-cache behind
the API module without touching screens, provided screens never read disk.

## Consequences

- Foundation STEP wires NetInfo in the shell, not in Auth or Storefront.
- i18n must include the no-internet copy (DF8).
- UI / Design System (1.7) inventories the dark full-screen + white pill as a shell pattern.
- A later downloads/offline phase adds a cache at the API module; it does not relocate the
  gate.
