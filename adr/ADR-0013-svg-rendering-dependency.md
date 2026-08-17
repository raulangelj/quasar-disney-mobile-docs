# ADR-0013: Adopt `react-native-svg`, and complete doc 03's dependency list

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- architecture/03-architecture-overview.md §7 (build vs. buy — hard dependencies)
- architecture/07-ui-design-system.md §6, §7, §9
- architecture/assets/README.md
- architecture/02-phasing-roadmap.md §10 (wordmark platform divergence, OQ-13)
- ADR-0012 (bundled font)

## Context

Session 1.7 needed an icon set and went looking for what the project already depends on. That surfaced a gap between two earlier decisions:

- **STEP-1.2** authored the placeholder brand and mock artwork as **SVG** — `assets/brand/*.svg` (four wordmark/strip files) and `assets/placeholder-art/*.svg` (seven tiles at 2:3, 16:9, 3:4).
- **STEP-1.3** listed the hard dependencies as React Native, Redux Toolkit, React Navigation, Styled Components, axios, redux-persist, `react-native-encrypted-storage`, and `@react-native-community/netinfo` (doc 03 §7).

React Native cannot render SVG without `react-native-svg`. As written, **the assets decided in 1.2 cannot be displayed by the stack decided in 1.3** — every tile in the storefront and every wordmark in the auth flow.

Reviewing the list also showed two further omissions: React Navigation requires `react-native-screens` and `react-native-safe-area-context` as peers, and doc 07 §7 makes safe-area insets load-bearing (the app renders no native headers, and Android 15 enforces edge-to-edge). Neither appears in doc 03.

The choice was not only "add the library": exporting every asset to `@1x/@2x/@3x` PNG would keep the dependency list unchanged.

## Decision

1. **Adopt `react-native-svg`** as a hard Phase-1 dependency. Placeholder artwork and icons render as SVG.

   **Rationale.** The assets already are SVG; the artwork must scale across a 375–440 pt device range (doc 07 §7) and the icons must re-tint from theme tokens per state and per mode — both of which PNG variants would have to enumerate as files, breaking criterion A1. It is autolinked and needs no manual native configuration.
   **Alternatives.** *All-PNG export* — no new dependency, but multiplies asset files per state/theme and forfeits free scaling. *`react-native-vector-icons`* — solves only icons, needs per-platform font linking, and ships a whole icon font for 13 glyphs.
   **Reversibility.** Medium — swapping to PNG later is an asset-pipeline change, not a code-architecture one.

2. **Wordmarks additionally ship as `@1x/@2x/@3x` PNG.** The wordmark SVGs use `<text>` with a font stack that falls back on Android (doc 02 §10, OQ-13); rasterizing removes the font dependency entirely rather than relying on an outlining pass being done correctly under deadline.

3. **Doc 03's hard-dependency list is completed** with `react-native-svg`, `react-native-screens`, `react-native-safe-area-context`, and `react-i18next` + `i18next` (doc 07 §9).

4. **Rule going forward:** a session that decides an *asset format* also names the dependency that renders it. This gap existed for three days across two accepted docs because nobody owned the seam between "we chose SVG assets" and "we listed our libraries".

## Consequences

**Easier.** The 13-icon set becomes essentially free (doc 07 §6) and theme-tinted by construction. Artwork scales to any device width with no `@Nx` matrix. The safe-area work doc 07 §7 requires is now backed by a listed dependency rather than an assumed one.

**Harder.** Four more npm dependencies in a project that had been deliberately lean, and four more things that can go wrong in the scaffold window that RISK-0003 already flags as the top schedule risk. All four are autolinked and widely used, but `pod install` is still `pod install`.

**Risk recorded.** RISK-0012 tracks the general failure mode — decisions that imply dependencies nobody adds to the list — with the check-in as the revisit trigger.
