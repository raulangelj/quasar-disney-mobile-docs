# ADR-0011: Theme is brand × surface mode, and modes are named by role

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- architecture/07-ui-design-system.md §5
- architecture/02-phasing-roadmap.md §4 (criteria A1, A2), §6 (DF4)
- architecture/01-system-overview.md §3 (central theme tokens)
- ADR-0001 (modular monolith; `shared/` kernel)

## Context

The reference screens carry **two distinct surfaces**: a near-black app theme for everything after login, and a light white-sheet-on-gradient theme for the whole MyDisney auth flow (`inputs/ui/streaming-reference-screens.md`). Doc 02 decision 9 already committed to carrying both inside one token structure rather than one palette with per-screen exceptions.

Two launch criteria depend on how that structure is shaped:

- **A1** — a second test theme exists, and switching to it re-skins **both** surface modes without touching a single component.
- **A2** — zero hardcoded colors, typography, or spacing outside the theme.

The obvious naming for two surfaces is `light` and `dark`. That naming is a trap. Every React Native developer knows `useColorScheme()`, and a theme whose modes are called light and dark invites exactly one future change: wiring them to the OS appearance setting. The moment that happens, a reviewer whose phone is in dark mode opens the app and the **auth flow renders dark** — a screen that is light in the reference for brand reasons, not device reasons. The bug would be introduced by someone doing something that looks obviously correct.

## Decision

1. **The theme has two axes:** a *theme* (brand token values — `qcplus`, plus the `ember` test theme) and a *surface mode* (`app` | `auth`). Modes live **inside** the theme: `Theme.modes: Record<SurfaceMode, ModeTokens>`. Every mode exposes an identical set of token keys.

   **Rationale.** A1 requires swapping a theme to re-skin both modes at once. If modes sat beside themes rather than inside them, a theme swap would only reach one surface.
   **Reversibility.** Low cost to extend (a third mode is a key); high cost to invert (every consumer changes).

2. **Modes are named for their role — `app` and `auth` — never `light`/`dark`.**

   **Rationale.** The name is the guardrail. `mode="auth"` reads as "this is the auth surface"; nobody wires that to an OS appearance setting. `mode="light"` reads as an invitation to.
   **Alternatives.** `light`/`dark` (rejected: invites the `useColorScheme()` bug); `surfaceA`/`surfaceB` (rejected: meaningless at the call site).

3. **`useColorScheme()` is never called in Phase 1.** A device in light mode sees an identical app.

4. **Mode is provided at the navigator boundary**, by a `ModeProvider` that is a nested `ThemeProvider` flattening `theme.modes[mode]` onto the active theme. No Redux state holds the mode.

   **Rationale.** The auth navigator is the auth surface — that is the same fact, so it should have one source. State could drift from the route; a provider at the boundary cannot.
   **Alternatives.** A `ui.mode` slice (rejected: two sources of truth for one fact); per-screen mode props (rejected: every new screen can forget).

5. **Components read `theme.colors.*` and never know which mode they are on.** A component that branches on mode is a defect, not a pattern.

6. **The A1 test theme is specified in the design system, not invented later:** `ember`, with every value contrast-checked to the same WCAG AA floor as `qcplus`.

## Consequences

**Easier.** A1 becomes a mechanical property rather than an aspiration — it is demonstrable by long-pressing the wordmark at sign-off (doc 07 §5). Adding a locale-, brand-, or client-specific skin later is a new theme file with no component changes. Auth and storefront can be built in parallel against the same token keys, which is what lets Dev A and Dev B work without coordinating (doc 02 §9).

**Harder.** Every token must be defined in **both** modes even where only one surface uses it, or a component moved between surfaces breaks. Semantic colors therefore carry two values each (doc 07 §2.3). Reviewers must reject `mode ===` branches in components, since the type system cannot forbid them.

**New work.** `ModeProvider`, the `DefaultTheme` augmentation, the `ember` theme file, and the dev swap affordance — all in the foundation STEP.

**Foreclosed in Phase 1.** OS-driven dark mode, a user-facing theme setting, per-screen theme overrides, and persisting a theme choice (which would also contradict ADR-0003's persist-auth-slice-only rule).

## Amendment (2026-08-17 — STEP-1.14 / ADR-0019)

`ThemeProvider` is **Emotion's** (`@emotion/react`), and styled primitives are `@emotion/native`.
Decision 4's `ModeProvider` shape is unchanged. Type the theme by augmenting `@emotion/react`'s
`Theme`, not styled-components' `DefaultTheme`.

