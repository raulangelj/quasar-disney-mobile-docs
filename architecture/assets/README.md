# Architecture assets

Design assets produced by the method and owned by the architecture docs. Session **1.7 (UI /
Design System)** formally adopts them and defines how the theme references them.

## `brand/` — the Dinsey- placeholder brand

quasar-disney-mobile is a POC modelled on Disney+ reference screens
(`../../inputs/ui/disney-plus-reference-screens.md`). **No Disney mark or artwork may appear in the
codebase or in any build.** These files are the substitution, decided 2026-08-14 (STEP-1.2).

| File | Use |
|------|-----|
| `dinsey-wordmark-light.svg` | Wordmark, white — dark surfaces (welcome screen, app chrome) |
| `dinsey-wordmark-dark.svg` | Wordmark, near-black — the light auth sheet |
| `dinsey-mark.svg` | Compact mark, 1:1 with its own gradient plate — app icon, tab bar, small surfaces |
| `brand-strip.svg` | The welcome screen's sub-brand row |

**Sub-brands** in the strip — PIXL, NOVA, ORBIT, WILDLENS, STREAMLY, SPORTX — are invented names,
deliberately *not* lookalikes of the real studios in the reference. Only the parent brand is a
lookalike, and that was an explicit project decision.

> **⚠️ "Dinsey-" is one letter from "Disney".** Confusing similarity is exactly what trademark law
> targets. Accepted for an internal, unpublished POC (see `RISK-0005` in
> `../../registries/risks.yml`). **Rename before any public release, app store submission, public
> site, or marketing material.**

### Known limitation — text is not outlined

The wordmarks and strip use SVG `<text>` with a font stack (`Avenir Next` → `Futura` →
`Trebuchet MS` → `Verdana` → `sans-serif`). Avenir Next resolves on iOS; **Android will fall back**,
so the brand renders differently per platform — a real problem for a demo whose subject is visual
fidelity.

**Fix before the 2026-08-18 sign-off** (~30 min): open each file in Figma or Inkscape, convert text
to outlines, re-export. Alternatively ship the wordmarks as `@1x/@2x/@3x` PNGs. Until then the
geometry — swoosh, dash, proportions, plate — is final and platform-independent; only the letterforms
are at risk.

## `placeholder-art/` — mock content artwork

Abstract key art at the reference aspect ratios, resolving OQ-09. Each is a self-contained SVG
(gradient, geometry, bottom scrim so overlay text stays legible).

| File | Ratio | Feeds |
|------|-------|-------|
| `tile-2x3-a.svg`, `-b.svg`, `-c.svg` | 2:3 | Standard portrait carousel |
| `tile-16x9-a.svg`, `-b.svg`, `-c.svg` | 16:9 | Continue-watching and live carousels |
| `tile-3x4-hero.svg` | 3:4 | Hero carousel (Phase 2) |

Three variants per ratio is enough that a visible row does not obviously repeat. Need more: copy a
file and shift the gradient stops — the compositions are parameterised only by colour.

Every tile carries a **bottom scrim**, so titles, progress bars, and badges laid over the lower
third stay readable without per-tile tuning.
