# ADR-0012: Bundle Inter rather than use the platform system font stack

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- architecture/07-ui-design-system.md §2.4, §13
- architecture/02-phasing-roadmap.md §1 (visual fidelity as an explicit goal), §10 (wordmark platform divergence)
- architecture/01-system-overview.md ("both platforms shown side by side")
- architecture/15-native-app-architecture.md §1 (bare RN, no Expo)

## Context

The default and cheapest choice in React Native is the system font stack: **SF Pro** on iOS, **Roboto** on Android. It costs nothing, links nothing, and gives native Dynamic Type behavior for free.

It also renders **two different typefaces** — different letterforms, widths, and vertical metrics. Doc 01 states the demo is shown with "both platforms side by side", and doc 02 §1 raises visual fidelity from implicit to an explicit sign-off goal. Under those two facts, a system stack means the two devices on the table do not match, in every string on every screen.

This project already has one instance of exactly this failure: doc 02 §10 records that early placeholder wordmark SVGs used `<text>` with an `Avenir Next` font stack that resolves on iOS and falls back on Android, and calls it "a real defect" on a demo whose subject is visual fidelity. Choosing a system stack would generalize that defect from one asset to the whole UI.

The reference material is an iOS app using a custom brand font, so matching it with a *platform* font was never going to be exact on either device.

## Decision

1. **Bundle Inter** (SIL Open Font License) and use it on both platforms for all UI text.

   **Rationale.** Identical rendering on iOS and Android; a large x-height and open apertures that hold up at the 10–12 pt `caption`/`micro` sizes where most of this app's text lives; and real tabular figures, so "11 min restantes" does not shift as the number changes.
   **Alternatives.** *System stack* — rejected above. *Manrope* — closer to the wordmark's geometry, but measurably less legible at caption sizes, which is where the metadata rows are. *Avenir Next* — the reference-adjacent choice and the family the wordmarks specify, but a licensed Monotype/Apple font, absent on Android and not redistributable.
   **Reversibility.** High. Family is a token; swapping it is one theme file and a re-link.

2. **Ship named weight files** (`Inter-Regular`, `-SemiBold`, `-Bold`, `-ExtraBold`) and reference them by family name, not numeric `fontWeight`.

   **Rationale.** On Android, numeric `fontWeight` against a bundled font silently synthesizes or ignores weights. Naming files is the only reliable mapping.

3. **Type tokens carry `fontFamily` per weight**, so a component never spells a weight or a family.

4. **This is recorded as a deliberate deviation from platform convention** (doc 07 §13), alongside press feedback. Behavior follows the platform; appearance follows the reference.

## Consequences

**Easier.** Screenshots from the two devices are comparable, which is what the sign-off session is doing. The type ladder's measured sizes render as designed on both. Criterion A2 gets simpler — one family name in one place.

**Harder.** ~350 KB of font files and a linking step (`react-native.config.js` + `npx react-native-asset`) in the scaffold, which is the schedule's highest-risk window (RISK-0003). Dynamic Type still works, but through RN's font-scaling rather than the system font's own optical sizing, which is why the `maxFontSizeMultiplier` caps in doc 07 §7 exist.

**Related.** This does not fix the wordmark defect — the wordmarks are SVG `<text>`, a separate asset problem tracked as OQ-13 and addressed by PNG export in ADR-0013. It does mean the wordmark is the *only* remaining place where text can render differently per platform.
