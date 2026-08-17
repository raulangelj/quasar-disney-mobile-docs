# ADR-0001: Modular monolith with a composition-root shell

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/03-architecture-overview.md`
- `architecture/02-phasing-roadmap.md` (DF5)
- `architecture/01-system-overview.md`

## Context

The app is a React Native stakeholder demo (iOS + Android) with two features that must be
implementable in parallel (auth and storefront) and extractable into separate repos later.
A backend is explicitly out of scope. The temptation is either a single “screens” blob or
premature extraction into packages/services.

## Decision

1. **One React Native app** (modular monolith). Five in-process modules: app shell, auth
   feature, storefront feature, shared kernel, API module.
2. **The app shell is the composition root** and may import every module to register
   screens, reducers, and middleware.
3. **Features never import each other** — only `shared/`. Shared never imports `features/`.
4. **Shared stays one component**, internally foldered (`theme/`, `ui/`, `i18n/`,
   `analytics/`). No types package; no npm workspaces in Phase 1.

**Rationale.** DF5 already forbids cross-feature imports; the two-developer split depends on
that seam. Folder splits give a later extraction path without package ceremony under the
2026-08-18 cutoff. A global types module would become a junk drawer every feature depends on.

**Alternatives.** Microservices / a separate mock server — rejected; we do not operate a
backend. Extracting auth/storefront/shared as packages now — rejected; extra versioning
with no independent deploy. Splitting shared into first-class modules (types, theme,
components) — rejected as too fine for Phase 1.

**Reversibility.** Medium. Folders can become packages later without rewriting screens if
the import rules hold.

## Consequences

- Auth and storefront STEPs can proceed on disjoint directories.
- The shell is the one place that knows the full graph; that is intentional, not a leak.
- Feature extraction later is a packaging change, not a rewrite — provided the rules are
  enforced in review.
