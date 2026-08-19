# Doc 07 — UI / Design System

**Version:** v0.3.2
**Status:** Draft
**Last updated:** 2026-08-19 (STEP-6.3)
**Audience:** Mobile developers, QA, stakeholders reviewing the 2026-08-18 demo

> The visual foundations of the React Native client — tokens, components, navigation, theming, accessibility, i18n, and motion — with exact values, so Phase 1a can be built without re-deciding any of it.

## Table of Contents

1. [Design principles](#1-design-principles)
2. [Tokens](#2-tokens)
3. [Components](#3-components)
4. [Navigation](#4-navigation)
5. [Theme](#5-theme)
6. [Iconography](#6-iconography)
7. [Responsive and device strategy](#7-responsive-and-device-strategy)
8. [Accessibility](#8-accessibility)
9. [Internationalization](#9-internationalization)
10. [Motion](#10-motion)
11. [Data visualization](#11-data-visualization)
12. [Implementation stack](#12-implementation-stack)
13. [Platform conventions](#13-platform-conventions)

---

## 1. Design principles

**Cinematic · immersive · art-forward.**

Artwork dominates; chrome recedes. Near-black surfaces lifted slightly off pure black, translucent chrome, a single accent, and a tight row rhythm so several carousels stack in one viewport. This is the direction the reference screens actually embody (`inputs/ui/streaming-reference-screens.md`), and visual fidelity is an explicit Phase-1 goal (doc 02 §1), not a nice-to-have.

What the principle costs, and where it is bounded:

- Translucent grey-on-dark chrome is where contrast fights you. Every text value in §2 was measured, and the reference's dimmest greys (~4:1) were **raised** rather than reproduced.
- Tight density is bounded by touch targets: visual density and hit density are independent (§8).

**Rejected directions:** *crisp/systematic* (better for a design-system argument, worse for fidelity), *premium/restrained* (lower lookalike exposure, less recognizable), *vivid/playful* (furthest from the reference).

## 2. Tokens

All values are React Native density-independent points (unitless in code). Every color pair listed as text has a measured WCAG 2.1 contrast ratio.

### 2.1 Color — theme `qcplus`, mode `app` (dark)

| Token | Value | Contrast | Use |
|-------|-------|----------|-----|
| `surface.base` | `#0E0E12` | — | Screen background |
| `surface.raised` | `#17171D` | — | Overlay, toast, alert surface |
| `surface.chrome` | `rgba(255,255,255,.07)` | — | Tab bar, filter pills |
| `border.hairline` | `rgba(255,255,255,.12)` | — | Chrome edges, elevation |
| `text.primary` | `#FFFFFF` | 18.9:1 | Titles, tile names |
| `text.secondary` | `#A1A1AA` | 7.52:1 | Episode line, metadata |
| `text.tertiary` | `#8A8A93` | 5.63:1 | Time-remaining, inactive icons |
| `accent` | `#38BDF8` | 8.99:1 | Watched progress, links on dark, active state |
| `accent.hover` / `accent.press` | `#7DD3FC` / `#0284C7` | — | Interaction ramp |
| `live` | `#FF4D63` | 5.96:1 | Live badge + live progress bar |
| `chip.fill` / `chip.text` | `#3F3F46` / `#D4D4D8` | 7.07:1 | Rating chip |

### 2.2 Color — theme `qcplus`, mode `auth` (light)

| Token | Value | Contrast | Use |
|-------|-------|----------|-----|
| `gradient.top` → `gradient.bottom` | `#0F3C46` → `#0A0A0E` | — | Welcome / auth backdrop |
| `surface.sheet` | `#FFFFFF` | — | The sheet over the gradient |
| `text.primary` | `#111114` | 19.0:1 | Headings |
| `text.secondary` | `#52525B` | 7.73:1 | Body copy, hints |
| `field.fill` / `field.placeholder` | `#E9E9EB` / `#5F5F68` | 5.21:1 | Text + password fields |
| `cta.fill` / `cta.label` | `#0E0E12` / `#FFFFFF` | 19.3:1 | Auth CTAs |
| `link` | `#1D4ED8` | 6.70:1 | Inline links |
| `error` | `#C81E1E` | 5.74:1 | Field underline + message |

### 2.3 Semantic ramp (both modes)

| Role | Mode `app` | Mode `auth` | Phase-1 use |
|------|-----------|-------------|-------------|
| `success` | `#4ADE80` (11.05:1) | `#15803D` (4.99:1) | none — reserved |
| `warning` | `#FBBF24` (11.54:1) | `#B45309` (5.13:1) | none — reserved |
| `error` | `#F87171` (6.96:1) | `#C81E1E` (5.74:1) | **auth inline error (F2)**, storefront error state |
| `info` | `#38BDF8` (8.99:1) | `#0369A1` (5.74:1) | none — reserved |
| `live` | `#FF4D63` (5.96:1) | n/a | 1b live carousel |

`live` is **not** an alias of `error`. They are both red and mean opposite things; aliasing them means retuning the error red silently changes the live badge.

**Rule — accent is never a button fill.** White text on `#38BDF8` is 2.14:1 (fail). Accent is a progress-bar / link / active-state color; pill CTAs on dark are white-fill-with-dark-label, as the reference does it.

### 2.4 Typography

Family: **Inter**, bundled, both platforms. Line heights are absolute (RN `lineHeight` is a number, not a multiplier).

| Token | Size | Line | Weight | Tracking | Used by |
|-------|------|------|--------|----------|---------|
| `display` | 30 | 36 | 800 | −0.02em | Welcome headline |
| `h1` | 26 | 31 | 800 | −0.015em | Auth sheet heading |
| `h2` | 21 | 26 | 700 | −0.01em | Screen title, row header |
| `h3` | 16 | 21 | 700 | 0 | Tile title, CTA label |
| `body` | 14 | 20 | 400 / 600 | 0 | Sheet copy, field text |
| `caption` | 12 | 16 | 400 | 0 | Metadata, error text, hints — **tabular figures** |
| `micro` | 10 | 13 | 700 | +0.06em | Chips, badges, uppercase CTAs |

Derived from the reference's measured sizes, not a modular ratio (1.25 would have produced a 33 pt `h1`, which the reference does not do). **No mono family** — nothing in Phase 1 would import it.

Weights ship as **named files** (`Inter-Regular/-SemiBold/-Bold/-ExtraBold`), because numeric `fontWeight` is unreliable for bundled fonts on Android. Linked via `react-native.config.js` + `npx react-native-asset`.

### 2.5 Spacing

4 pt base scale: `space.xxs` … `space.xxxxxxxxxl` (4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 72, 80, 88, 96, 104). Screen-specific layout constants stay in the feature (e.g. welcome poster fan), not in `layout.*`. Role aliases only where a component author must not re-derive: `layout.gutter`, `layout.tileGap`, `layout.rowGap`.

**Density — reference-tight:**

| Role | Value |
|------|-------|
| `layout.gutter` (screen edge) | 16 |
| `layout.tileGap` | 8 |
| `layout.rowGap` (between row groups) | 20 |
| metadata line gap | 6 |
| auth sheet padding | 20 |
| auth sheet height ratio | 0.78 |
| field / CTA height | 48 |

An 8-pt-only grid was rejected: the reference's real values include 8, 12, and 20.

### 2.6 Shape & elevation

| Token | Value | Applies to |
|-------|-------|------------|
| `radius.chip` | 4 | Rating chip, badge |
| `radius.field` | 8 | Text / password field |
| `radius.tile` | 10 | All carousel tiles |
| `radius.surface` | 16 | Overlay, toast, alert |
| `radius.sheet` | 24 | Auth sheet top corners |
| `radius.full` | 999 | Pill CTA, filter pill, icon button, play button |

The mix is deliberate — large on the sheet, small on content, full on actions.

**Elevation is `surface.base` → `surface.raised` plus `border.hairline`, in both modes. No shadows**, with one exception: the auth sheet's top edge over the gradient (`0 -8px 24px rgba(0,0,0,.18)`), the only place a surface genuinely sits over a lighter ground.

Why: on `#0E0E12` a drop shadow is invisible — there is nothing lighter behind it. Using shadows only in light mode would make the two modes express depth by *different mechanisms*, which breaks criterion A1's "swap tokens, touch no component". Two implementation facts reinforce it: RN's iOS (`shadowOffset`…) and Android (`elevation`) shadow APIs do not match visually, and Android will not render a shadow on a view with `overflow: hidden`, which every rounded tile needs.

Translucent chrome is `rgba(255,255,255,.07)`, **not** a backdrop blur — RN needs a native blur dependency for an effect invisible at 7% over near-black. Phase-2 polish.

### 2.7 Motion tokens

| Token | Duration | Easing | Used by |
|-------|----------|--------|---------|
| `motion.instant` | 100 ms | ease-out | Press in/out, toggles |
| `motion.quick` | 180 ms | ease-out | Overlay fade, error appear |
| `motion.base` | 260 ms | ease-in-out | Navigator cross-fade, sheet |
| `motion.slow` | 400 ms | ease-in-out | Reserved — unused in Phase 1 |
| `motion.shimmer` | 1400 ms | linear, loop | Skeleton sweep |
| `press.scale` / `press.opacity` | 0.965 / 0.7 | — | Tiles / everything else |

## 3. Components

**Ownership rule:** a component lives in `shared/components/` if **two or more** of {auth, storefront, shell} render it; otherwise it lives with its feature. This makes the foundation STEP's deliverable a closed list and resolves doc 02 §9's "shared atom nobody owns" collision hazard.

**Feature module layout (all features):** **`architecture/03-architecture-overview.md` §8.1.1** is the canonical template (auth on disk). Doc 07 §3 covers component ownership. **Storefront migrates in STEP-6.4**; **new features copy auth from day one**. Do **not** add files under `features/*/ui/` or `screens/*/components/`.

| Path | Purpose |
|------|---------|
| `features/<feature>/api.ts` | RTK `injectEndpoints` when the feature owns API operations. |
| `features/<feature>/screens/<ScreenName>/` | One directory per route or screen flow; entry file `<ScreenName>.tsx`. Screen-local layout constants may sit beside the entry (e.g. `welcomeLayout.ts`). **No `components/` subfolder under screens.** |
| `features/<feature>/components/atoms/` | Feature-only atoms — not in `shared/components/` because only this feature renders them. |
| `features/<feature>/components/molecules/` | Composed feature UI (e.g. `CredentialsForm`, `PortraitTile`). |
| `features/<feature>/components/organisms/` | Larger feature sections (e.g. `WelcomeHero`, `AuthSheetLayout`, `HomeFeedList`). |
| `features/<feature>/components/index.ts` | Barrel exports for feature components. |
| `features/<feature>/helpers/` | Pure utilities — not under `screens/` or `components/`. **Tested** units use `helpers/<name>/` (doc 03 §8.1.1). |
| `features/<feature>/hooks/` | Data / pagination / composition hooks (storefront; omit if unused). **Tested** units use `hooks/<name>/`. |
| `features/<feature>/screens/index.ts` | Navigator-facing screen exports only. |
| `features/<feature>/assets/` | Optional feature-local static media. |
| `features/<feature>/README.md` | Feature tree documented; keep aligned with disk. |
| `shared/components/{atoms,molecules,organisms}/` | Cross-feature UI. Components with co-located `<ComponentName>.styles.ts` or `*.test.ts` use **`shared/components/<tier>/<ComponentName>/`** (doc 03 §8.1.1). Simple atoms/icons stay flat files. |

Example (auth — **actual on disk**, STEP-6.2):

```
features/auth/
  README.md
  api.ts
  assets/welcome/              # PNG posters + wordmark; index.ts exports fan config
  components/
    atoms/
      AuthFooterBrandRow.tsx
      AuthGradientBackground.tsx
      BrandStrip.tsx
      SubBrandWordmark.tsx
    molecules/
      CredentialsForm.tsx
    organisms/
      AuthSheetLayout.tsx
      WelcomeHero.tsx
    index.ts
  helpers/
    emailValidation.ts
    isApiError.ts
  screens/
    WelcomeScreen/
      WelcomeScreen.tsx
      welcomeLayout.ts
    Login/
      EmailEntryScreen.tsx
      PasswordEntryScreen.tsx
    PlaceholderScreen/
      PlaceholderScreen.tsx
    index.ts
  state/
    slices/auth/
      authSlice.ts
      authSlice.test.ts
    actions/
      logout.ts
    selectors/
      auth.ts
```

Example (storefront — **target** STEP-6.4):

```
features/storefront/
  README.md
  api.ts
  components/
    molecules/
      PortraitTile.tsx
      ProgressTile.tsx
    organisms/
      CarouselRow.tsx
      HomeFeedList.tsx
    index.ts
  helpers/
    placeholderArt/
      placeholderArt.ts
      placeholderArt.test.ts
      index.ts
    variantConfig/
      variantConfig.ts
      variantConfig.test.ts
      index.ts
    progressTileLabel/
      progressTileLabel.ts
      progressTileLabel.test.ts
      index.ts
  screens/
    HomeScreen/
      HomeScreen.tsx
    ComingSoonScreen/
      ComingSoonScreen.tsx
    index.ts
  hooks/
    types.ts
    index.ts
    composeHomeContainers/
      composeHomeContainers.ts
      composeHomeContainers.test.ts
      index.ts
    usePaginatedContainers/
      usePaginatedContainers.ts
      usePaginatedContainers.test.ts
      index.ts
    useCarouselPage/
      useCarouselPage.ts
      useCarouselPage.test.ts
      index.ts
    useComposedHome/
      useComposedHome.ts
      silentReload.test.ts
      index.ts
    useSilentContinueWatchingReload.ts
```

Tested helpers and hooks use a **unit subdirectory** (`<name>/<name>.ts` + `<name>.test.ts` + optional `index.ts`). Simple untested utilities stay flat — see doc 03 §8.1.1.

**Redux state (features with slices):** when a feature owns Redux state, everything lives under `state/` — not beside it.

| Path | Purpose |
|------|---------|
| `features/<feature>/state/slices/<slice>/` | Slice definition + unit tests for that slice. |
| `features/<feature>/state/actions/` | Imperative store writers that dispatch slice actions (e.g. auth `logout`). |
| `features/<feature>/state/selectors/` | One file per slice — all selectors for reading that slice from the store. |

Auth is the reference implementation (STEP-6.2). Storefront has **no Redux slice in Phase 1a** (feeds live in RTK Query cache + `hooks/`). When storefront — or any future feature — adds a slice, **use the same `state/` tree as auth**; do not colocate selectors on the slice file or actions at the feature root.

**STEP-6.4 storefront migration checklist** — before UI polish, align folder layout with auth (doc 03 §8.1):

1. `ui/` → `screens/<ScreenName>/` (screen entry files only).
2. Feature UI → `components/{atoms,molecules,organisms}/` at feature root (classify per atomic design — **not** under screen dirs).
3. Move pure utilities from `ui/` → `helpers/`.
4. Add `components/index.ts` and `screens/index.ts`; update shell/tab imports.
5. Delete empty `ui/`. **No new files under `features/*/ui/`** after 6.4.
6. Update `features/storefront/README.md` to the post-migration tree.
7. **State:** omit `state/` until a slice exists; then use auth's `state/slices|actions|selectors` tree.

### 3.1 Atoms — `shared/components/atoms` (foundation STEP)

| Component | Variants / states |
|-----------|-------------------|
| `Text` | One per type token |
| `Button` | `tone: onDark \| onLight` · `variant: solid \| ghost` · `cornerRadius: pill \| cta` · `size: md(48) \| sm(36)` · `loading` · `disabled` · `pressed` |
| `IconButton` | Circular translucent; 44 pt `hitSlop`; back / overflow |
| `Chip` | Rating copy (`7+`, `13+`, `16+`, `ATP`) |
| `Badge` | `tone: live \| label \| provider` |
| `ProgressBar` | `tone: watched \| live`; `value 0..1`; track always rendered |
| `TextField` | empty / focused (2 px ring) / filled / **error** (2 px bottom underline + message above the hint) |
| `PasswordField` | secure / revealed; label changes with state |
| `Link` | `tone: onDark (accent) \| onLight (blue)` |
| `Divider`, `Spinner` (26/16), `Skeleton` (takes the tile ratio) | — |
| `Artwork` | `ratio: 2:3 \| 16:9 \| 3:4`; scrim baked into the asset; skeleton fallback |
| `Wordmark` | `tone: light \| dark` |
| `FilterPill` | `form: logo \| iconLabel`; `selected` (Phase 2 usage, atom built now) |

**Button API:** one component with `tone` as an explicit prop, not derived from theme mode — the welcome screen puts an `onDark` button over the gradient while the sheet below is light, and a mode-derived tone gets that screen wrong.

### 3.2 Molecules

| Component | Lives in | Phase |
|-----------|----------|-------|
| `SectionHeader`, `TabBarItem`, `ErrorState`, `EmptyState` | `shared/components/molecules` | 1a |
| `PortraitTile` (2:3, art only) | `features/storefront/components/molecules` | 1a |
| `ProgressTile` (16:9 + play + bar + meta block) | `features/storefront/components/molecules` | 1a |
| `LiveTile`, `LandscapeTile` | `features/storefront/components/molecules` | 1b |
| `HeroCard` | `features/storefront/components/organisms` | Phase 2 chrome; 1a renders the hero container as a **3:4 portrait stand-in** (OQ-24 closed) |
| `AuthSheetLayout`, `WelcomeHero` | `features/auth/components/organisms` | 1a |
| `CredentialsForm` | `features/auth/components/molecules` | 1a |
| `AuthGradientBackground`, `BrandStrip`, `SubBrandWordmark`, `AuthFooterBrandRow` | `features/auth/components/atoms` | 1a |

### 3.3 Organisms

`Container` (one config-driven carousel for every variant — DF6 / ADR-0007) and `HomeFeedList` in `features/storefront/components/organisms`; `AppHeader`, `TabBar`, `LoadingGate`, `NoInternetOverlay` in the shell.

### 3.4 Screen states

| State | Treatment |
|-------|-----------|
| Cold-start gate | Wordmark + spinner; blocks until `/me` + HomeFeed + CW resolve (doc 04 §6) |
| Storefront loading | **Skeletons at real tile geometry** — no layout jump on arrival |
| `loadMore` | Trailing skeleton tile in the row |
| Auth submit | Spinner inside the button, label hidden, width held |
| Feature error | `ErrorState` + retry — **not** the no-internet overlay (doc 15) |
| Empty | CW with zero resources: the row is **omitted**, not rendered empty |

### 3.5 Deliberately not built

Modal/dialog (card tap is a native `Alert` — DF7), toast/snackbar, table, date picker, avatar component, search field, settings list. Listed so their absence is a decision, not an oversight.

## 4. Navigation

```
RootNavigator                       switches on selectIsAuthenticated
├─ AuthNavigator      stack · mode auth  · headerShown: false
│   ├─ Welcome
│   ├─ EmailEntry
│   └─ PasswordEntry  (owns the inline error state, F2)
└─ AppTabsNavigator   tabs  · mode app   · custom tab bar
    ├─ Home           the only live tab
    ├─ Novedades  ┐
    ├─ Buscar     ├─ ComingSoon placeholder screen
    └─ Perfil     ┘
NoInternetOverlay · LoadingGate — shell-level, outside both navigators
```

Two navigators rather than one guarded stack: an unauthenticated user has **no route** to the storefront because it is not mounted, and the surface mode is chosen at the navigator boundary rather than per screen.

| Decision | Call |
|----------|------|
| Inert tabs | **Tappable → `ComingSoon` placeholder.** Refines doc 02's "inert": a tap that does nothing is indistinguishable from a bug in front of the sign-off audience. One screen, three routes, one i18n string. |
| Back navigation | Chevron, iOS edge-swipe, and Android hardware/gesture back **all pop the stack**. Android's back is not optional. Back from PasswordEntry **clears the password, keeps the email** (matching `(editar)`); back is **blocked during an in-flight submit**. |
| Headers | `headerShown: false` everywhere; screens render their own chrome. The reference has no native header on any screen, and a native header renders platform-natively (divergence). Cost: safe-area insets are ours (§7). |
| Tab bar | React Navigation `BottomTabBar` with a custom `tabBar` renderer — keeps route state and bottom inset, matches the reference's 4 unlabeled icons. |

**Not in Phase 1:** details route (DF7), deep links, modal routes, drawer, nested stack in the Home tab, navigation-state persistence.

## 5. Theme

**Two axes: theme (brand) × surface mode.** Modes are named for their **role** — `app` and `auth` — never `light`/`dark`. Appearance names invite wiring them to `useColorScheme()`, which would turn the auth flow dark on a device in dark mode; the sheet is light because the reference's sheet is light, not because the device asked.

```ts
interface Theme {
  name: 'qcplus' | 'ember';
  modes: Record<'app' | 'auth', ModeTokens>;
  type; space; radius; motion;      // mode-independent
  colors: ModeTokens['colors'];      // the ACTIVE mode, injected by ModeProvider
}
```

- `useColorScheme()` is **never called** in Phase 1.
- Mode is fixed at the navigator boundary via `ModeProvider` (a nested `ThemeProvider` that flattens `modes[mode]` onto the theme). No Redux state holds it, so it cannot drift from the route.
- Every mode exposes **the same token keys**, so a component reads `theme.colors.text.secondary` without knowing its surface. This is what makes criterion A1 mechanical.

**The A1 test theme is specified, not left to invention:** `ember` (warm graphite `#14100C`, amber accent `#FBBF24`, `#4A2C10` gradient, `#B45309` link), every value contrast-checked to the same AA floor (lowest pair 5.0:1). Rendered proof exists in the session working page.

**Swap trigger:** a **dev affordance** — long-press the wordmark on Welcome cycles the theme. A1 becomes demonstrable live at sign-off rather than a claim backed by a diff. Not a user setting (that would imply the settings screen doc 01 defers, and persistence that ADR-0003 forbids).

Artwork is **not** themed: it is content, and real key art will not be themed either.

## 6. Iconography

**13 hand-authored SVG icons** as RN components over `react-native-svg`, in `shared/components/icons/`: cast, download, home (filled + outline), bolt, search, profile, chevron-left, chevron-down, eye, eye-off, overflow, play. 24 pt grid, 1.7 stroke, `color` from theme tokens so active/inactive and both modes come free.

`react-native-vector-icons` was rejected: per-platform native font linking, a whole icon font for 13 glyphs, and none of its sets match the reference's specific glyphs. PNG icons were rejected: per-state and per-theme tinting would require file variants, breaking A1.

See ADR-0013 — the SVG assets decided in STEP-1.2 already required this dependency.

## 7. Responsive and device strategy

**Device range:** 375–440 pt phones, iOS and Android. No tablet layout, no foldables.

**Tile width is computed from a visible-count, not fixed:**

```
tileWidth = (screenWidth − gutter − tileGap × (v − 1)) / v
```

`v` = tiles visible in the rail, **configured per rail** with a per-variant default (1.9 landscape, 3.6 portrait, 1.15 hero). The peek — the partly-visible next tile that signals the row scrolls — *is the spec*, so it holds at every device width instead of being an accident of fixed widths. `v` is **client-side**, keyed by `variant` — it is **not** a `Container` wire field (**OQ-37** closed).

| Topic | Decision |
|-------|----------|
| Safe areas | `SafeAreaProvider` + `useSafeAreaInsets()`, applied in `AppHeader`, `AuthSheetLayout`, and the tab bar **only** — never per screen, so nothing double-pads. Load-bearing, since §4 removed native headers. |
| Orientation | **Portrait locked** both platforms (plist + manifest). Every reference screen is portrait; landscape carousels are a different grid. |
| OS font scaling | Honored, capped: `maxFontSizeMultiplier` **1.6** body and above, **1.3** caption, **1.2** micro. Uncapped, a 200% text size makes the tile metadata taller than its artwork. |
| Density | SVG art and icons scale freely; PNG wordmarks ship `@1x/@2x/@3x`. |

## 8. Accessibility

**Target: WCAG 2.1 AA.** Most of it was designed in rather than retrofitted.

| Requirement | Status |
|-------------|--------|
| Text contrast ≥ 4.5:1 | Met — every pair in §2 measured; the reference's ~4:1 greys raised |
| Touch targets ≥ 44 pt | Met — `hitSlop` on `IconButton`; visual density and hit density are independent |
| Text resize to 200% | Met, capped (§7) |
| Visible focus indicator | Met — 2 px ring on fields (external keyboards, Switch Control) |
| Non-text contrast ≥ 3:1 | Met — hairline borders and progress track clear 3:1 |
| Color not the only signal | Met — error is red **and** underline **and** message; live is red **and** a ⚡ badge |
| Reduced motion | Met — §10 |
| Name / role / value | Below |

**Screen-reader model — one tile is one element.** A `ProgressTile` is seven views; left ungrouped, VoiceOver makes each a stop (6 swipes per tile, 60 to cross a row, and nothing announcing it is tappable). Instead:

- The tile is `accessible` with `accessibilityRole="button"` and a **translated label template** (not concatenated strings): *"{name}. {episode}. Clasificación {rating}. Quedan {minutes} minutos, {percent} % visto."*
- Each **additional action** is a sibling element, not a child — an overflow button nested inside the grouped pressable is unreachable.
- Decorative artwork is hidden (`accessibilityElementsHidden` / `importantForAccessibility="no-hide-descendants"`).

| Surface | Rule |
|---------|------|
| Tab bar (unlabeled) | `accessibilityLabel` per tab + `accessibilityState={{selected}}`; placeholder tabs add a hint |
| Password toggle | Label changes with state (*Mostrar / Ocultar contraseña*) |
| **Auth inline error** | `accessibilityLiveRegion="assertive"` + `announceForAccessibility` — **without this a failed login is silent to a screen-reader user, i.e. criterion F2 fails for them** |
| Progress bar | Folded into the tile label as a percentage; not its own element |
| Loading | Skeletons hidden from the tree; the gate announces "Cargando" |
| No-internet overlay | `accessibilityViewIsModal` so focus cannot wander behind it |
| Section headers | `accessibilityRole="header"` for rotor navigation |

**Verification gap, stated plainly:** UI tests are deferred to 1b (doc 02), so there is no automated a11y assertion in 1a. The props are written *as the atoms are written* — near-free then, a full sweep later — and verified by a **10-minute VoiceOver + TalkBack pass** added to the Tuesday-AM smoke. Tracked as RISK-0011.

## 9. Internationalization

**Mechanism: `react-i18next`.** The app needs interpolation and plurals, not just lookup ("11 min restantes", the a11y templates above). A bespoke strings layer would also teach the wrong lesson in a repo whose purpose is to be a migration template. A hand-rolled typed module remains the documented fallback if the Saturday schedule slips.

| Sub-decision | Call |
|--------------|------|
| Keys | **Semantic IDs** (`auth.password.title`), never English source strings |
| Locales | **`es-419` only** in Phase 1 — the reference material is Spanish (LatAm) |
| Detection | **Forced `es-419`**; no device-locale lookup. A demo that changes language with the reviewer's phone cannot be rehearsed. Detection is a Phase-3 line change |
| Dates / relative time | `Intl` via Hermes — no `moment` / `date-fns` |
| RTL | **Not enabled, not foreclosed:** `marginStart`/`paddingEnd`/`textAlign:"start"` throughout, never `Left`/`Right`. Costs nothing now; makes RTL a flag flip plus QA later |
| Layout tolerance | No fixed-width text containers. The reference-tight density assumes Spanish; German/Finnish would grow ~30% — a known constraint of the density choice |

**Row headers are data, not UI copy.** Doc 04 described `Container.name` as "i18n in the client", but a name arriving from a feed can only be translated if the wire sends a key. Decision: **Phase-1 mocks send display strings, rendered verbatim, and the real API must return text localized to the request's locale** — how content APIs actually behave. Doc 04 is corrected accordingly; the contract obligation is **OQ-30** for session 1.11.

## 10. Motion

Durations and easings are in §2.7. Behaviors:

| Interaction | Motion |
|-------------|--------|
| Tile press | Scale 0.965 over `instant`, native driver |
| Button / pill / link / icon press | Opacity 0.7 over `instant` |
| **Login (auth → app)** | **Cross-fade over `base`**, running *after* the boot gate resolves — it fades from the auth sheet to a **painted** storefront, never to a spinner |
| Auth stack | **Platform defaults** — iOS slide-from-right, Android fade-through. Required for the iOS edge-swipe to drag the screen it dismisses (§4) |
| Skeleton | `shimmer` sweep |
| Progress bar | Animates to value over `base` on mount, then only on data change |
| No-internet overlay | Fade `quick` in and out — connectivity flaps, and a hard-cut overlay strobes |

Press animations use `useNativeDriver: true`; `transform` and `opacity` qualify, which is why those two were chosen over an animated border or background color — anything animating layout would stutter on the JS thread exactly while the carousel scrolls.

**Reduced motion — shorten or substitute, never remove:**

| Animation | Reduce Motion on |
|-----------|------------------|
| Tile press | Opacity 0.7 — feedback kept, transform dropped |
| Login transition | Cross-fade shortened to `instant` |
| Stack transitions | `animation: 'fade'` — no lateral movement |
| Skeleton | Static `surface.raised` fill |
| Progress bar | Renders at value immediately |

Read via `AccessibilityInfo.isReduceMotionEnabled()` + a change listener, exposed as a `useReducedMotion()` hook in `shared/`. Components never query the API directly — the same discipline as tokens. Removing press feedback entirely would leave a user with no confirmation their tap registered: a worse outcome than the animation.

## 11. Data visualization

**Not applicable.** The only quantitative display is the continue-watching progress bar, which is a token'd atom (§3.1), not a chart. No charting library, no visualization palette. Recorded so a later reader knows it was considered and dismissed.

## 12. Implementation stack

**Emotion (`@emotion/native` styled + `@emotion/react` `ThemeProvider`) + TypeScript**, per ADR-0019.

```ts
// shared/theme/ModeProvider.tsx — mode is a nested Emotion ThemeProvider that flattens the active mode
import { ThemeProvider, useTheme } from '@emotion/react';

export const ModeProvider = ({ mode, children }) => {
  const base = useTheme();
  const value = useMemo(() => ({ ...base, ...base.modes[mode] }), [base, mode]);
  return <ThemeProvider theme={value}>{children}</ThemeProvider>;
};
```

```ts
import styled from '@emotion/native';

const Title = styled.Text`
  color: ${({ theme }) => theme.colors.text.primary};
  font-size: ${({ theme }) => theme.type.h2.size}px;
  padding: ${({ theme }) => theme.space.m}px;
`;
```

| Concern | Decision |
|---------|----------|
| Layout | `shared/theme/{tokens/, themes/qcplus.ts, themes/ember.ts, ModeProvider.tsx, types.ts, styled.d.ts}` |
| Typing | Augment `@emotion/react`'s `Theme` in `emotion.d.ts` — otherwise `theme` is `any` inside every template, defeating the overview's "typed values throughout" |
| **A2 enforcement** | An **ESLint rule, not code review**: `no-restricted-syntax` banning hex literals and `rgba(` outside `shared/theme/`, plus a rule banning `styled-components` imports. Styled primitives come from `@emotion/native`. A1/A2 are launch criteria and should fail a build, not depend on a reviewer noticing. Raw spacing numbers stay a review item — a numeric rule is too fragile to be worth it |
| Token consumers outside Emotion styled | React Navigation's theme object, `StatusBar`, and the SVG icon `color` prop must all read from the same tokens. These three are where a hardcoded value usually survives an otherwise clean migration |

## 13. Platform conventions

**Rule: follow platform convention for *behavior*, follow the reference for *appearance*.** Behavior is what users' hands expect and what breaks when fought; appearance is what the demo is judged on.

| Area | Call | Detail |
|------|------|--------|
| Back navigation | Follow | Android hardware/gesture + iOS edge-swipe both pop (§4) |
| Screen transitions | Follow | Platform defaults; the one place the demo deliberately differs across devices |
| Status bar | Follow | `light-content` in both modes — the bar is over dark pixels on every screen; translucent on Android |
| Keyboard | Follow | `KeyboardAvoidingView` `behavior="padding"` (iOS) / `"height"` (Android). The auth sheet must keep the CTA reachable with the keyboard up |
| Alert on card tap | Follow | Native per-OS dialog — correct, since it is a placeholder for navigation (DF7) |
| Splash / icon | Follow | Static launch screen from `qc-plus-mark`; no animated splash library |
| **Press feedback** | **Deviate** | Scale + opacity, not Material ripple — so the side-by-side matches |
| **Typography** | **Deviate** | Bundled Inter, not SF Pro / Roboto (ADR-0012) |
| Haptics | Skip | Nothing in Phase 1 warrants a dependency |

**Android 15 (targetSdk 35) enforces edge-to-edge:** system bars go transparent and content draws underneath whether the app opts in or not. The safe-area handling in §7 is therefore correctness on Android, not polish — worth knowing before the scaffold day, given RISK-0003.

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Design principles | Cinematic · immersive · art-forward | Visual fidelity is an explicit Phase-1 goal | Crisp/systematic, restrained, or playful directions |
| 2 | Color | `#0E0E12` neutral-cool base, `#38BDF8` accent, teal auth gradient; two modes | Closest to the transcribed reference; neutral greys keep art from fighting chrome | Teal-tinted, indigo, and pure-black alternatives |
| 3 | Accent usage | Never a button fill (2.14:1 with white) | Accent stays progress/link/active; CTAs invert per surface | An accent-filled primary button |
| 4 | Typography | **Bundled Inter**, 7-step ladder from measured reference sizes | Identical on both platforms; holds up at caption/micro sizes (ADR-0012) | System-font Dynamic Type; SF Pro / Roboto |
| 5 | Spacing | 4 pt scale, reference-tight density (gutter 16 / gap 8 / rows 20) | ~2.5 rows per viewport — the storefront reads full | Comfortable density; 8-pt-only grid |
| 6 | Touch targets | 44 pt via `hitSlop`, independent of visual density | Reference glyphs are ~20 pt; AA floor is non-negotiable | Enlarging glyphs to reach the minimum |
| 7 | Shape | Reference radii (tile 10 / sheet 24 / field 8 / chip 4 / full) | Deliberate contrast: large sheet, small content, full actions | Uniformly soft or uniformly crisp systems |
| 8 | Elevation | Surface lift + hairline border, both modes; no shadows (one sheet exception) | Shadows are invisible on near-black; RN shadow APIs diverge across platforms | Material elevation; shadow-based cards |
| 9 | Components | 14 atoms / 9 molecules / 8 organisms; `shared/components` iff ≥2 consumers | Closes doc 02 §9's shared-atom collision hazard | Per-feature duplicate atoms |
| 10 | Button API | One `Button` with explicit `tone` | Tone cannot be derived from mode — Welcome breaks that assumption | Separate Primary/Secondary/Ghost components |
| 11 | Press feedback | Tiles scale 0.965; everything else opacity 0.7 | Opacity is a weak signal on dark artwork | Material ripple; uniform opacity |
| 12 | Loading | Gate spinner · row skeletons at true geometry · in-button spinner | Each matches what is being awaited; no layout jump (doc 04 §6) | One-size spinner or skeletons everywhere |
| 13 | Navigation | Two root navigators; tabs; no native headers | Unauthenticated users have no route to mount; mode set at the boundary | Single guarded stack; native headers |
| 14 | Inert tabs | Tappable → `ComingSoon` placeholder | A dead tap reads as a bug at sign-off | Refines doc 02's "inert" |
| 15 | Back navigation | Chevron + iOS swipe + Android back all pop; blocked mid-submit | Android back is not optional | Gesture-disabled auth flow |
| 16 | Theme structure | theme × surface mode; modes named by **role** (ADR-0011) | Prevents `useColorScheme()` from breaking the light auth flow | OS-driven dark mode in Phase 1 |
| 17 | Theme swap | Dev long-press on the wordmark; `ember` test theme specified | Makes A1 demonstrable live at sign-off | A user-facing theme setting |
| 18 | Icons | 13 hand-authored SVG components over `react-native-svg` | Token-tinted, no font linking; dependency already required (ADR-0013) | `react-native-vector-icons`; PNG icons |
| 19 | Tile sizing | Visible-count formula; `v` is **client-side**, keyed by variant (**OQ-37**) | Peek is the spec, so it holds at every width; rails stay tunable | Fixed widths; magic viewport fractions; a wire `visibleCount` |
| 20 | Device strategy | 375–440 pt phones, portrait locked, insets in 3 components, font-scale caps | Every reference screen is portrait; no native header to pad for us | Tablets, landscape, foldables |
| 21 | Accessibility | WCAG 2.1 AA; one tile = one element + sibling actions; live-region errors | Carousels fail screen readers by default; a silent error fails F2 for those users | Automated a11y assertions in 1a (RISK-0011) |
| 22 | i18n | `react-i18next`, semantic keys, forced `es-419`, RTL-safe properties | Plurals/interpolation genuinely needed; template repo should show the production answer | Device-locale detection; shipped RTL |
| 23 | Row-header text | Wire sends **localized display strings**; client renders verbatim | Row names are data, not UI copy | Corrects doc 04; contract obligation OQ-30 |
| 24 | Motion | 4-step scale; cross-fade login after the boot gate; platform stack transitions | Native-driver-only properties; never fade into a spinner | A cinematic through-black transition |
| 25 | Reduced motion | Shorten or substitute, never remove | Removing feedback is a worse a11y outcome than the animation | — |
| 26 | Data-viz | N/A — no charts | The progress bar is an atom | A charting dependency |
| 27 | Implementation | **Emotion** (`@emotion/native` + `@emotion/react` ThemeProvider); `ModeProvider` flattens the mode (**ADR-0019**) | Components never know their surface — A1 becomes mechanical | `styled-components`; a second theme context |
| 28 | A2 enforcement | ESLint bans hex/`rgba(` outside `shared/theme/` | A launch criterion should fail a build, not rely on review | A numeric spacing rule (too fragile) |
| 29 | Platform conventions | Behavior follows platform; appearance follows the reference | Two deliberate deviations recorded (press feedback, typography) | Full Material compliance on Android |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-37~~ | ~~Does `visibleCount` travel on the wire as a `Container` field?~~ **Resolved (1.14):** client-side, keyed by variant. **Not OQ-28** (that ID is the sign-off binary owner). | — | closed |
| ~~OQ-38~~ | ~~No Phase-1a screen exposes logout — add a hidden affordance, or expiry-only?~~ **Resolved (1.14):** logout in 1a is **expiry-only** (Perfil is ComingSoon; no hidden logout). **Not OQ-29** (that ID is the reachability probe). | — | closed |
| OQ-30 | Confirm the real API returns row `name` localized to the request's locale (and how locale is conveyed) | Backend team | Phase 3 / OQ-34 |

**OQ-24** is closed (1.14): 1a ships a 3:4 hero stand-in. **OQ-22** is closed (1.11) — wire names are in `architecture/11-interface-contracts.md` §7; note that **`Card.name` is now `Card.title`**, and that doc 11 §7.3 records why `Container.name` is rendered verbatim while `error.message` never is.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.7 | Initial design system from the UI session. Tokens, components, navigation, theme, icons, device strategy, a11y, i18n, motion, implementation, platform conventions. ADR-0011, ADR-0012, ADR-0013. Opened OQ-28/29/30; opened RISK-0011/0012. Corrected doc 04's client-side row-name i18n; surfaced three missing dependencies in doc 03. |
| v0.1.1 | 2026-08-17 | STEP-1.11 | Closed OQ-22: wire names live in doc 11 §7 (`Card.name` → `Card.title`). Doc 11 §7.3 records the rendered-verbatim vs never-rendered rule for server strings; §8.2 keys all error copy off `error.code`, not server prose. |
| v0.2.0 | 2026-08-17 | STEP-1.14 | §12: **Emotion** (`@emotion/native` + `@emotion/react` ThemeProvider) replaces styled-components (**ADR-0019**). Token model unchanged. |
| v0.2.1 | 2026-08-17 | STEP-1.14 | Closed OQ-24 (3:4 hero stand-in). `visibleCount` is client-side (**OQ-37**). Logout in 1a is expiry-only (**OQ-38**). Those questions were mis-numbered as OQ-28/29. |
| v0.2.2 | 2026-08-18 | STEP-2.4 | Closed **OQ-13**: brand SVG `<text>` converted to paths in `quasar-disney-mobile-app` `src/shared/assets/brand/` (runtime source); hub `architecture/assets/brand/` remains provenance. |
| v0.2.3 | 2026-08-18 | STEP-6.2 | Default brand theme slug **`qcplus`** (legacy placeholder slug retired). Added `radius.cta` (10 pt) for welcome/offline pill CTAs. `Button` gains optional `cornerRadius: pill \| cta`. QC+ wordmark assets replace legacy placeholders; welcome i18n uses **QC+** / **QC Entertainment**. |
| v0.2.4 | 2026-08-18 | STEP-6.2 | Stakeholder welcome pack wired: poster fan + PNG wordmark, violet 3-stop gradient (`gradient.mid`), QC+ palette (`#0A0A1F → #150C2E → #050410`, `#F7F5FF` text/CTA, `#9AC4FF` links, `#FF8A3D` accent). Headline and brand strip removed from welcome layout. |
| v0.2.5 | 2026-08-18 | STEP-6.2 | Spacing scale renamed to semantic steps `space.xxs` … `space.xxxxxxxxxl`; welcome-specific layout constants moved out of `layout.*` into the auth welcome screen module. |
| v0.2.6 | 2026-08-18 | STEP-6.2 | Auth feature UI reorganized: `features/auth/ui/` → `features/auth/screens/` with per-screen directories (`WelcomeScreen/`, `Login/`), module `components/`, and feature-level `helpers/`. |
| v0.2.7 | 2026-08-18 | STEP-6.2 | **Feature screen layout** documented as mandatory for all features (`screens/<ScreenName>/` entries only; `helpers/` at feature root). Storefront `ui/` is legacy — migrate in STEP-6.4. Superseded by v0.3.0 (no `screens/*/components/`). |
| v0.2.8 | 2026-08-18 | STEP-6.2 | **Feature Redux layout** (auth reference): `state/slices/<slice>/`, `state/actions/`, `state/selectors/` — mandatory for future feature slices. |
| v0.2.9 | 2026-08-18 | STEP-6.2 | STEP-6.4 storefront migration checklist: mirror auth `screens/` + `helpers/` layout; `state/` tree required when a feature gains a slice. |
| v0.3.0 | 2026-08-18 | STEP-6.2 | Feature **`components/{atoms,molecules,organisms}/`** at module root — not under `screens/`. Auth migrated; storefront + new features must follow. |
| v0.3.1 | 2026-08-18 | STEP-6.2 | Canonical feature module template moved to **doc 03 §8.1.1** with full auth tree on disk; doc 07 cross-references it. |
| v0.3.2 | 2026-08-19 | STEP-6.3 | Auth sheet chrome: `layout.authSheetHeightRatio` (0.78), scrollable sheet body, **MiQC+** sub-brand slot (`SubBrandWordmark`), email footer hairline + grey `AuthFooterBrandRow`. i18n: `common.subBrand` → **MiQC+**; `auth.email.*` / `auth.password.*` use **QC+** / **QC Entertainment** placeholders. |
| v0.3.4 | 2026-08-19 | — | **Tested unit subdirectories** for `helpers/` and `hooks/` documented; storefront tree updated (doc 03 §8.1.1). |
| v0.3.3 | 2026-08-19 | STEP-6.4 | Storefront migrated to auth-parity layout (`screens/`, `components/`, `helpers/`). Home chrome polish: header icon tertiary, tab inactive tertiary, section headers gutter-aligned, hero tile hairline border, `layout.rowGap` vertical rhythm. Grep gate pass on living artifacts. |
