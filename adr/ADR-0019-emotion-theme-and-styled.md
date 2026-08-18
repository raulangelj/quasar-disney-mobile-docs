# ADR-0019: Emotion for ThemeProvider, palette, and styled components

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/07-ui-design-system.md` §5, §12
- `architecture/03-architecture-overview.md` §7
- `overview.md` (kickoff constraint was Styled Components)
- ADR-0011 (theme × surface mode; `ModeProvider` is a nested `ThemeProvider`)

## Context

Kickoff and docs 01/03/07 locked **Styled Components** (`styled-components/native`) as the styling
library, with a `ThemeProvider` and token palette in `shared/theme/`. The product intent is the
same — a theme context, a color palette, typography/spacing tokens, and styled primitives — but
the library the team wants is **Emotion**, which also ships a `styled` API. Keeping
`styled-components` would train the migration template on the wrong package.

## Decision

1. **Styling and theming are Emotion.** `@emotion/native` provides `styled` (View, Text, Pressable,
   …). `@emotion/react` provides `ThemeProvider` and `useTheme`. Do not add `styled-components`.

2. **The token model in doc 07 is unchanged.** Palette, type ladder, spacing scale, two surface
   modes (`app` / `auth`), and the `ember` test theme stay in `shared/theme/`. `ModeProvider` remains
   a nested `ThemeProvider` that flattens `theme.modes[mode]` onto the active theme (**ADR-0011**).
   Only the package behind `ThemeProvider` / `styled` changes.

3. **Type the theme.** Augment `@emotion/react`'s `Theme` (not styled-components' `DefaultTheme`)
   so `theme` inside a styled template is not `any`.

4. **A2 enforcement follows the import.** ESLint bans `styled-components` and non-native CSS-in-JS
   entry points. Styled primitives come from `@emotion/native`. Hex / `rgba(` literals stay banned
   outside `shared/theme/`.

**Rationale.** Emotion gives the ThemeProvider + palette + styled-component workflow the team
already designed, without a second CSS-in-JS runtime. Tokens, modes, and A1/A2 do not depend on
which library flattens the theme.

**Alternatives.** Keep `styled-components/native` — rejected; it is the kickoff default, not the
intended stack. Restyle / StyleSheet-only — rejected; criterion A1 needs a theme context.
`@emotion/css` or web Emotion without `@emotion/native` — rejected; this is a React Native app.

**Reversibility.** Medium. Every styled file imports Emotion; swapping back is mechanical but
touches every component. The token files do not.

## Consequences

- Doc 07 §12, doc 01/03 hard-dependency lists, and `overview.md` name Emotion.
- ADR-0011 stands: modes, `ModeProvider`, and "components never know their mode" are unchanged.
- Responsive behaviour stays doc 07 §7 (visible-count tile formula, font-scale caps). Emotion does
  not replace that with a separate scaling library.
