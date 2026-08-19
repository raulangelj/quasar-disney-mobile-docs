# Input — Disney+ reference screens (iOS, es-419)

**Source:** Stakeholder-supplied screenshots of the production Disney+ iOS app, pasted in chat
during STEP-1.2 (2026-08-14). Locale: Spanish (LatAm). Device: iPhone, 1320×2868 px (@3x → 440×956 pt).
**Intent stated by the user:** *"Intentemos que sea lo más parecido a Disney, adjunto las imágenes
de las pantallas a recrear."* — visual fidelity to these screens is a goal, not just pattern demo.

> **Status of the image files.** The screenshots arrived as chat attachments; the binaries are not
> in the repo yet. Drop them into this folder as `01-home-para-ti.png` … `06-home-logged-in.png`
> (naming below) so later sessions and fresh chats can see them. This file is the written
> transcription meanwhile.
>
> **Password screenshot — redaction status.** The original capture showed a plaintext password;
> the user redacted it (2026-08-14) and the corrected version is the one to commit. The **email
> address is still visible** in the redacted version — clear it too before committing if the
> repo is shared beyond the team. No credential is recorded in this file.

> **Legal note.** These are references to a third-party product for UX-pattern purposes. Disney
> marks, logos, and artwork are not ours to ship — the demo must use placeholder branding and
> mock artwork. See the "Trademark & asset substitution" note at the end.

## Screen inventory

| # | Suggested filename | Screen | Flow position |
|---|--------------------|--------|---------------|
| 1 | `01-home-para-ti.png` | Home "Para ti" — hero carousel variant | Post-login |
| 2 | `02-login-password-error.png` | MyDisney password entry — **error state** | Auth step 3 |
| 3 | `03-login-password-empty.png` | MyDisney password entry — empty state | Auth step 3 |
| 4 | `04-login-email.png` | MyDisney email entry | Auth step 2 |
| 5 | `05-welcome-landing.png` | Disney+ welcome / landing | Auth step 1 (app entry) |
| 6 | `06-home-logged-in.png` | Home "Para ti" — continue-watching + rows | Post-login |
| 7 | `07-no-internet.png` | No-internet full-screen gate | Overlay on any flow |

**Implied navigation order:** 5 → 4 → 3 → (2 on failure) → 6 / 1. Connectivity loss at any point → 7; reconnect restores auth or storefront from session state.

---

## 1. Welcome / landing (`05`)

- Full-bleed **dark teal→near-black vertical gradient**; a faint, blurred content collage sits
  behind the whole screen at low opacity.
- **Top:** a row of 5 floating content tiles, staggered in size and vertical offset (center tile
  largest, flanking tiles progressively smaller/lower) — portrait ~2:3 cards with rounded corners
  and a subtle cyan/blue glow border on the two "Plan Premium" tiles, which carry a small
  `Plan Premium` pill badge at the bottom-left of the tile.
- **Center:** Disney+ wordmark logo, then a two-line headline in bold white
  ("Las mejores películas y series más los deportes en vivo de ESPN").
- **Brand strip:** monochrome logos separated by `+` — Disney · Pixar · Marvel · Star Wars ·
  National Geographic · hulu · ESPN. Marvel is red, hulu green, ESPN red; the rest white.
- **Below:** small grey caption ("Podrás crear y administrar tu cuenta en") with a **cyan link**
  (`disneyplus.com/next`).
- **CTA:** full-width **white** button, black uppercase letter-spaced label `INICIAR SESIÓN`,
  fully rounded corners (~pill, radius ≈ 8–10 pt here — squarer than the auth CTAs).
- **Footer:** two lines of small grey copyright text, centered.
- No status-bar-adjacent chrome; content is vertically centered in the lower two-thirds.

## 2. Email entry (`04`)

- **Top third:** same teal gradient, back chevron in a **circular translucent button** at
  top-left, Disney+ wordmark centered.
- **Body:** a **white sheet** with large top corner radius (~24–28 pt) covering the bottom ~78%.
  This sheet-over-gradient is the signature auth layout.
- Inside the sheet, left-aligned:
  - **MyDisney** wordmark (black).
  - Large bold heading `Escribir correo para continuar` (~30 pt, tight leading, wraps to 1–2 lines).
  - Body copy in dark grey, 2 lines.
  - **Text field:** light grey (`#E9E9EB`-ish) filled rect, no border, radius ~8 pt, placeholder
    `Correo electrónico` in mid-grey, generous height (~72 pt).
  - **CTA:** full-width **black pill** button (radius = height/2), white bold `Continuar`.
  - **Hairline divider**, then a bold small heading
    ("Disney+ es parte de la Familia de Compañías Walt Disney"), a grey explainer paragraph,
    and a **row of 7 greyed brand logos** (Disney, abc, ESPN, Marvel, Star Wars, hulu, National Geographic).

## 3. Password entry — empty (`03`)

Same shell as `04`. Differences:

- Heading `Escribir contraseña`.
- Body copy states the email to sign in with; the address is **bold** and followed by a
  **blue `(editar)` link** inline.
- **Password field:** same grey filled rect, placeholder `Contraseña`, with an **eye-with-slash
  toggle** at the right edge (slashed = hidden).
- Grey hint below the field: `Distingue mayúsculas y minúsculas.`
- Black pill CTA `Iniciar sesión`.
- **Blue link with a chevron-down:** `Más información sobre MyDisney ▾` (expander).
- **Blue link:** `¿Problemas para iniciar sesión? Enviar un código de acceso único`.

## 4. Password entry — error (`02`)

Same as `03`, plus the error treatment — this is the state that defines our validation pattern:

- The field keeps its grey fill but gains a **2 px red bottom border** (underline only, not a
  full outline).
- The typed value is **visible** (eye toggle un-slashed).
- A **multi-line red error message** sits directly under the field, above the case-sensitivity
  hint. It is long-form prose (4 lines), not a terse one-liner.
- The `Distingue mayúsculas y minúsculas.` hint stays in grey below the error.
- Everything below shifts down; the CTA and links are unchanged.

## 5. Home "Para ti" — hero variant (`01`)

- **Background:** near-black (`#0E0E12`-ish), slightly lifted from pure black.
- **Header row:** large bold `Para ti` title at left; at right, a **cast icon** and a
  **downloads icon** (tray with down-arrow, rounded-square outline).
- **Filter pill rail** (horizontally scrollable): pills with dark translucent fill, thin light
  border, fully rounded. Contents mix **logo-only** pills (Disney+, hulu, ESPN) and
  **icon + label** pills (`🎬 Películas`, `📺 Series`). Logo pills are wider and use the brand
  wordmark in its brand color (hulu green, ESPN white-on-dark).
- **Hero carousel:** a single large **portrait-ish card (~3:4 / 4:5)** occupying most of the
  viewport width, with the **previous and next cards peeking** at both edges (~15 px sliver each).
  Rounded corners ~12 pt, thin light border on the active card.
  - Full-bleed key art; **provider badge** (hulu) top-right.
  - Bottom-centered overlay stack: a small white **`Nueva película` pill** (dark text), the
    **title treatment as artwork** (not a text label), a bold white tagline, a white
    **`Ver ahora`** action line, then a metadata row: **`13+` rating chip** (grey rounded rect) ·
    `2026` · `Drama` separated by middots.
- **Below the hero:** section header `Recomendaciones para ti` (bold, ~22 pt), then a row of
  **portrait 2:3 tiles**, two visible with the third clipped.
- **Bottom tab bar:** translucent dark, 4 items — home (filled house, active/white), a
  **lightning bolt** (novedades), **search** (magnifier), and a **profile avatar** (circular
  character image). Inactive items are grey; no labels.

## 6. Home "Para ti" — rows variant (`06`)

Same header + tab bar as `01`, but no filter pill rail and no hero — this screen is the
**row-stack** layout, and it's the one that defines most of our carousel variants:

- **`Continuar viendo`** — **16:9 landscape tiles**, each with:
  - a **circular translucent play button** centered on the art,
  - a **cyan progress bar** pinned to the tile's bottom edge (partial fill),
  - below the tile: `11 min restantes` in grey, with a **`⋮` overflow menu** at the row's right,
  - then bold white title, then a metadata line: **rating chip** (`7+`, `16+`) followed by
    `T3:E12 El Univers-araña - Parte 1`.
- **`Series y especiales de Marvel`** — **portrait 2:3 tiles**, no overlay text; a
  `Disney+ ORIGINAL` lockup is baked into some artwork. Third tile clipped at the edge.
- **`Destacados en vivo y próximamente`** — **16:9 tiles** with:
  - a red **`⚡ VIVO` badge** bottom-left over the art,
  - a **red progress bar** on the tile's bottom edge (live position),
  - below: `Comenzó hace 7 h 26 min` in grey, bold title, then `2026 • ATP` metadata.

---

## Patterns to extract (for sessions 1.3 / 1.4 / 1.7)

**Carousel variants visible in these 6 screens (feeds the "up to 8 variants" decision in doc 01):**

| Variant | Tile aspect | Distinguishing chrome |
|---------|-------------|-----------------------|
| Hero / spotlight | ~3:4, near-full-width, neighbors peek | badge pill, title art, CTA line, metadata row |
| Continue watching | 16:9 | play button, cyan progress bar, time-remaining, `⋮` menu, episode line |
| Standard portrait | 2:3 | none (art only) |
| Live / upcoming | 16:9 | `VIVO` badge, red progress bar, relative-time line |
| Standard landscape | 16:9 | title below (implied; not shown in these captures) |

**Cross-cutting UI primitives (atoms/molecules):**
rating chip · provider badge · label pill (`Nueva película`) · live badge · progress bar
(cyan = watched, red = live) · filter pill (logo and icon+label forms) · circular icon button ·
overflow menu · pill CTA (black-on-white and white-on-black) · inline text link (blue on light,
cyan on dark).

**Two distinct themes, not one:**
- **Dark app theme** — near-black surfaces, white/grey text, cyan accent (`#0EA5E9`-ish),
  translucent chrome. Used by all post-login screens.
- **Light auth theme** — white sheet on teal gradient, black CTAs, blue links, red error.
  Used by the whole MyDisney flow.
This has direct consequences for the "central theme tokens" success criterion in
`architecture/01-system-overview.md` §3: the token set must carry **two surface modes**, not one
palette.

**Auth flow is 3 screens + states, not 1:** welcome → email → password → error. Doc 01 §4 lists
"Login screen (email/password)" as a single item; these references expand it.

## 7. No internet (`07`) — captured STEP-1.3a (2026-08-16)

Full-screen **dark** gate (near-black, same family as the app theme — not the light auth sheet).
Centered copy + pill CTA; no header, tabs, or logo.

- Body (white, centered): `Es necesario revisar tu conexión a internet. Volveremos a cargar automáticamente la pantalla una vez que se establezca la conexión.`
- CTA: full-width **white pill**, black uppercase `REINTENTAR`.
- Behavior implied by copy: auto-reload when connectivity returns; Retry is the manual path.
- On restore, the app resumes **auth or storefront according to session state** — this is a shell-level overlay, not a feature screen.

Substitute branding N/A (no marks). Copy goes through i18n (DF8); Spanish is the reference locale.

## Trademark & asset substitution

The demo cannot ship Disney/Marvel/Star Wars/hulu/ESPN marks or real key art. Reproduce the
**layout, spacing, motion, and component structure**; substitute:
- brand wordmarks → the project's own placeholder wordmark(s),
- provider/filter pills → generic labels,
- key art → licensed-free or generated placeholder imagery at the same aspect ratios.
This is a Phase-1 constraint to record in the phasing doc, not an open question.
