# Doc 02 — Phasing & Roadmap

**Version:** v0.4.2
**Status:** Draft
**Last updated:** 2026-08-17 (planning session)
**Audience:** Product stakeholders, mobile developers, backend team, QA

> How quasar-disney-mobile is cut into phases: what Phase 1 delivers by the 2026-08-18 stakeholder
> sign-off, what is deliberately deferred, and the constraints Phase 1 must not foreclose.

## Table of Contents

1. [Phase 1 as a functional POC with visual fidelity](#1-phase-1-as-a-functional-poc-with-visual-fidelity)
2. [Phase 1a, the 2026-08-18 demo cut](#2-phase-1a-the-2026-08-18-demo-cut)
3. [Phase 1b, completing Phase 1](#3-phase-1b-completing-phase-1)
4. [Launch criteria](#4-launch-criteria)
5. [Later phases](#5-later-phases)
6. [Don't-foreclose list](#6-dont-foreclose-list)
7. [Phase dependencies and critical path](#7-phase-dependencies-and-critical-path)
8. [Schedule and schedule risk](#8-schedule-and-schedule-risk)
9. [Two-developer split](#9-two-developer-split)
10. [Placeholder brand](#10-placeholder-brand)

---

## 1. Phase 1 as a functional POC with visual fidelity

**Milestone type:** POC / stakeholder demo — *not* an MVP. No end user derives value from it; its
job is to prove the architecture and the visual result to an internal audience.

**Goal (one sentence):**

> Ship a React Native app running on iOS and Android whose login → storefront flow recognizably
> reproduces the Disney+ experience (with our own branding and artwork), built on the full
> Throughstone stack — atomic design, custom hooks, Redux Toolkit + RTK Query, Emotion theme
> tokens, typed models, and a swappable API boundary — so stakeholders sign off on both the
> pattern and the visual result.

**What changed from the System Overview.** Doc 01 §4 listed *"Login screen (email/password)"* as one
scope item and §5 estimated *3–4 days*. Stakeholder reference screenshots supplied during this
session (`inputs/ui/streaming-reference-screens.md`) show the auth flow is **three screens plus a
distinct error state**, that the app carries **two surface themes** (dark app / light auth), and
that the home screen has **two layouts across five carousel variants**. Visual fidelity was also
raised from implicit to an explicit goal. **This document's schedule supersedes doc 01 §5's 3–4 day
estimate.**

**Fidelity is a goal, with a hard substitution rule.** We reproduce layout, spacing, component
structure, and interaction — never Disney's marks or artwork. Brand wordmarks, provider pills, and
key art are replaced with our own placeholders at identical aspect ratios. This is a Phase-1
constraint, not an open question.

**Phase 1 is split.** An immovable stakeholder sign-off on **2026-08-18** leaves ~3.5 working days
(Sat 15 – Mon 17 plus Tue 18 morning, weekend included by decision), against a ~6–7 day estimate for
full Phase 1 with tests. Rather than deliver everything half-finished — the worst outcome for a
demo whose subject *is* craft — Phase 1 splits into **1a** (demo-gated, hard date) and **1b** (the
remainder, no external date).

## 2. Phase 1a, the 2026-08-18 demo cut

### In scope

**Foundation**
- Bare React Native scaffold (no Expo) + TypeScript, running on iOS and Android.
- Modular feature-oriented file structure (`features/*`, `shared/*`).
- **Central theme with two surface modes** — dark (app) and light (auth) — as one token structure,
  not one theme with exceptions.
- Atoms and molecules used by the screens below: rating chip, label pill, badge, progress bar,
  pill CTA (black-on-white and white-on-black), text field with error state, password field with
  visibility toggle, inline text link, circular icon button, tile.

**Auth flow** (light theme)
- **Welcome / landing** — gradient background, tile collage, brand strip, primary CTA.
- **Two-step credentials** — email screen, then password screen, on the white sheet-over-gradient layout (recovered into 1a with the second dev; §9).
- **Inline error state** — red underline on the field plus multi-line red message below it,
  matching the reference's treatment. Driven by a *simulated failed fetch*, not a local branch.
- Hardcoded demo credentials; opaque token stored in the auth slice from day one.

**Storefront** (dark theme)
- Header (`Para ti` + cast/downloads icons) and 4-item bottom tab bar (home active; others inert — STEP-1.7 defines "inert" as **tappable, leading to a `ComingSoon` placeholder screen**, so a dead tap is not mistaken for a bug at sign-off; doc 07 §4).
- **Two carousel variants**, both config-driven:
  - *Continue watching* — 16:9 tiles, play button, cyan progress bar, time-remaining, `⋮` menu,
    title + rating chip + episode line.
  - *Standard portrait* — 2:3 tiles, art only.
- Mock content rendered from the API layer as **paginated pages**; **tap any card → alert with the title**, behind the
  same handler that will later navigate. Storefront hooks own `loadMore` / `hasMore` (DF12).

**Data & contracts**
- **API contract authored by us** — TypeScript interfaces, enums, response envelope, and error
  shape — plus the written contract document. We do not build a backend.
- **Mock API layer that simulates real fetches**: returns Promises with artificial latency and can
  be made to fail, so loading and error states are genuine rather than decorative.
- Redux Toolkit store + RTK Query `baseApi`; no component or hook touches `fetch` or `axios` directly.

**Tests**
- Every element gets a test: reducers, middleware, hooks, and the API/mock layer.
- **UI/component rendering tests are explicitly deferred** to 1b.

**Analytics**
- Hook exists with its final signature, backed by `console.log` stubs.

### Out of 1a (moved to 1b or later)

| Item | Lands in | Why deferred |
|------|----------|--------------|
| Live and landscape carousel variants | 1b | Config additions to an existing component — cheap later, not on the critical path. Data model parks `'live'` (doc 04) |
| Hero / spotlight **chrome** + filter pill rail | Phase 2 | Most expensive component in the reference. The feed already includes a `hero` container; 1a renders a **3:4 stand-in** (OQ-24 closed) |
| UI/component tests | 1b | Highest cost-to-signal ratio under the deadline; logic tests carry the testability argument |
| Formal QA smoke checklist | 1b | Distinct from the **release-build smoke** at every sync point (doc 09 §6.1), which *is* in 1a |

## 3. Phase 1b, completing Phase 1

No external date. Delivers: the live (`VIVO` badge, red progress bar) and landscape carousel
variants; UI/component rendering tests; the formal QA smoke checklist; and any polish deferred
under deadline pressure. **Two-step auth is in 1a** (§2 / §9).

## 4. Launch criteria

Phase 1a is done when all of the following are true.

**Functional**

| # | Criterion |
|---|-----------|
| F1 | On **both** iOS and Android (simulator or device), welcome → email → password → home completes end to end |
| F2 | Wrong credentials produce the **inline error state** from the reference — not a generic alert, not a crash — driven by a simulated failed fetch |
| F3 | Home renders both carousel variants from mock data; tapping any card shows an alert with the title |

**Architecture — what is actually being demonstrated**

| # | Criterion |
|---|-----------|
| A1 | **Theme swap verified**: a second test theme exists, and switching to it re-skins both surface modes without touching a single component |
| A2 | **Zero hardcoded** colors, typography, or spacing outside the theme |
| A3 | **API swap**: mocks and a future real backend pass through the same `baseApi` and axios instance; substituting the base URL and removing `axios-mock-adapter` is the entire change |
| A4 | **API contract document delivered** to the backend team |
| A5 | **Module extraction verifiable**: no feature imports from another feature — only from `shared/` |
| A6 | **Tests green** for every reducer, middleware, hook, and the API/mock layer |

**Process**

| # | Criterion |
|---|-----------|
| P1 | Stakeholder sign-off session held **2026-08-18** and passed |

Criteria A1–A5 trace directly to doc 01 §3's *API swap readiness*, *theme swap readiness*, and
*stakeholder sign-off* success criteria.

## 5. Later phases

### Phase 2 — complete storefront (no backend required)
- Hero / spotlight carousel — near-full-width card with neighbours peeking, badge pill, title
  artwork, CTA line, metadata row. *The Phase-1 **feed contract** already includes a hero row
  (doc 04 / ADR-0006); this phase is the full spotlight **chrome** if 1a shipped a stand-in
  (OQ-24 **closed 1.14:** 1a ships a 3:4 stand-in).*
- Filter pill rail (logo-only and icon+label forms). *Needs more than one mocked content source to
  mean anything.*
- **Content details screen** — metadata, description, cast, "similar to this" row; the alert-on-tap
  handler becomes real navigation. *Deferred because it is surface, not architecture; the alert
  already proves the interaction path.*
- Bitrise CI + installable iOS/Android builds for QA. *Blocked externally (accounts, signing), not
  technically; can start in parallel once the scaffold exists.*

### Phase 3 — backend integration
**We never build a backend.** This phase integrates the API the backend team delivers, against the
contract we authored in Phase 1a.
- Real endpoints replace the mock adapter; base URL swap plus adapter removal.
- Real JWT — same client shape (access token in the auth slice). Refresh, if the IdP issues it, plugs into the API module / interceptor, not screens (doc 16). No mock refresh in Phase 1.
- Search and the "novedades" tab. *Both need real content data; mocks would make them theatre.*
- Real analytics behind the existing hook signature.

*Gated externally on the backend team's delivery — the only external dependency in the roadmap.*

### Phase 4+ — streaming app
Video player / playback, multi-user household profiles, offline downloads, parental controls,
settings, cast, payments/subscriptions. *Each is a project in its own right; none changes a Phase-1
decision. Playback additionally depends on Phase 3 for real content URLs and DRM.*

## 6. Don't-foreclose list

Constraints Phase 1a must respect even though it doesn't yet exercise them. These become inputs to
sessions 1.3, 1.4, 1.7, 1.11, and 1.12.

| # | Constraint | What it forbids in Phase 1a |
|---|------------|------------------------------|
| DF1 | **Backend swap** | No component or hook calls `fetch` or `axios`. All I/O goes through **RTK Query** on `baseApi` (axios instance + interceptors inside the API module). Mocks are `axios-mock-adapter` on that instance and carry the *exact* shape expected of the real backend — field names, enums, response envelope, error shape |
| DF2 | **Simulated fetches, not local data** | The mock layer returns Promises with artificial latency and can fail on demand, so loading and error states are real code paths |
| DF3 | **JWT** | The auth slice holds an opaque token from day one, even though it's fake. Persist that slice only, into **`react-native-encrypted-storage`** (Keychain / EncryptedSharedPreferences) from day one (not AsyncStorage). Refresh still plugs in without touching screens |
| DF4 | **Re-skin by tokens** | Two surface modes live in the theme structure itself, not as per-screen exceptions |
| DF5 | **Feature extraction to repos** | No feature imports from another feature — only from `shared/`. The app shell is the composition root and may import features |
| DF6 | **Carousel variants** | The carousel is config-driven from the first commit. Adding hero, **progress**, live, and landscape must be adding configuration, not components — even though 1a ships only a subset. Hero and progress are **Container variants** (ADR-0007) |
| DF7 | **Details navigation** | The alert lives behind the same handler that will later navigate; the swap must not touch the tile components |
| DF8 | **i18n** | No loose hardcoded strings, even with no multi-language support in 1a. The reference material is Spanish |
| DF9 | **Analytics** | The hook ships with its final signature behind `console.log` stubs |
| DF10 | **Trademark substitution** | No Disney/Marvel/Star Wars/hulu/ESPN marks or real key art in the codebase or assets, at any phase |
| DF11 | **Connectivity gate** | No-network is a **shell overlay** (NetInfo + reference no-internet screen), not a feature fetch error and not an offline cache. Restore by auth state. **Online = interface up**, no reachability probe (ADR-0014). See doc 15 / ADR-0004 |
| DF12 | **Storefront pagination** | Home rows and cards **page**. Storefront wraps RTK Query queries with `{ items, loadMore, hasMore }`; mocks return pages. Screens do not call axios. **Opaque `nextCursor`** (doc 04). Two **`Container[]`** endpoints; cards in **`resources`** (ADR-0006 / ADR-0007). **Page sizes locked (1.11):** `limit` 16 on HomeFeed, 10 on Continue Watching and on `resources` (doc 11 §6.3). |

## 7. Phase dependencies and critical path

**Within Phase 1a the order is not free:**

1. Scaffold (RN bare + TypeScript + module structure)
2. **Theme tokens, both modes** — before any component. If colors are hardcoded first, criterion A1
   is lost and retrofitting costs more than doing it right
3. Atoms and molecules
4. **Auth flow** — the smallest surface that exercises the whole stack end to end
5. **Storefront** — dark theme, both carousel variants, feed endpoints / catalog cache

Auth precedes storefront deliberately: if the pattern is wrong, it surfaces on day 2, not day 5.

**Between phases:**
- Phase 2 depends only on Phase 1. *Exception:* Bitrise can start in parallel once the scaffold
  exists — its blocker is external.
- Phase 3 depends on the backend team's delivery. The Phase-1a contract exists to make that
  integration a swap rather than a rewrite.
- Phase 4 depends on Phase 3 for real content URLs and DRM.

**Architecture sessions compressed to fit.** With ~3.5 days total, only the sessions that the code
cannot proceed without run before the build: **1.3** (component boundaries), **1.7** (design system
— theme tokens and atomic inventory), **1.4** (content card schema), **1.11** (interface contracts —
promoted back into 1a because the contract now *defines* the mocks), and **1.12** (test strategy,
short, since tests are in the gate). Sessions **1.5** Scaling, **1.6a** Identity & Auth, **1.8**
Infrastructure, **1.9** Environments, and **1.10** Observability were originally **Deferred** — none
informs a 1a decision. **1.6** Security (no real PII), **1.13** Glossary, and **1.14** Cross-Cutting
Review run abbreviated.

*Amendment (2026-08-17):* **1.5**, **1.6a**, **1.8**, **1.9**, **1.10**, **1.11**, **1.12**, and
**1.13** subsequently ran. **1.14 Cross-Cutting Review** closes STEP-1. RISK-0002 is closed.

1.10 declined the whole telemetry stack for Phase 1 (no metrics, tracing, health checks,
dashboards, alerting, or vendor) and added exactly one thing to the build: a **shell-root error
boundary**, so a render exception on 18 Aug degrades to a recoverable screen rather than a white
one. It changes no schedule item. Accepted posture: **RISK-0014**.

1.8 scoped itself to the mobile build-and-distribute path (no server exists to host) and made
one change that touches this document's schedule: **the release build is the declared sign-off
artifact, not a fallback** — its ~30-minute first-time setup belongs at the **Tue 18 AM sync
point** in §8, alongside integration and smoke. Procedure: `runbooks/release-deploy.md`.

## 8. Schedule and schedule risk

| When | Work |
|------|------|
| Fri 14 Aug (remainder) | Sessions 1.3, 1.7 |
| Sat 15 AM | Sessions 1.4, 1.11, 1.12; abbreviated 1.6, 1.13, 1.14 to close STEP-1; then the planning session authors the Phase-1a STEPs |
| Sat 15 PM | **Dev A:** scaffold, theme, shared atoms · **Dev B:** contract + mock API layer *(parallel)* |
| Sat 15 midday | **Checkpoint** — scaffold running on both platforms? (RISK-0003) |
| Sat 15 EOD | **Sync point 1** — foundation and contract merge |
| Sun 16 – Mon 17 | **Dev A:** auth feature *(parallel, no coordination)* · **Dev B:** storefront feature |
| Tue 18 AM | **Sync point 2** — integration, theme-swap verification, polish, smoke on both platforms, **plus the release build + install on two devices** (`runbooks/release-deploy.md`; ~30 min first time) |
| **Tue 18** | **Stakeholder sign-off — immovable** |

See §9 for the two-developer split this table assumes.

**Top schedule risk: the bare React Native scaffold on two platforms.** Pods, signing, emulators,
and a misaligned Xcode can silently consume half a day. **Trigger:** if there is no app running on
both platforms by Saturday midday, the Sunday plan no longer closes. **Pre-agreed fallback:** drop
Android for Saturday, pursue iOS only, and bring Android up on Monday.

Other risks are inherited from doc 01 §7 and unchanged, except that *"3–4 day timeline too tight"*
is no longer a risk but a **realized constraint** this document responds to.

## 9. Two-developer split

**Two senior devs are confirmed for 15–18 Aug** (resolves OQ-07). This section defines how they work
in parallel without blocking each other.

### The method constraint that shapes this

`runbooks/collaboration.md`: **one owner per STEP; substeps are never split across people.** Two
people working simultaneously therefore requires **two STEPs with disjoint scope**, each on its own
`step-NNNN-*` branch. This is not bureaucracy — it is what makes "each at their own pace" possible,
because disjoint scope is what removes the need to coordinate.

### The seam

The parallelization seam already exists: **DF5 forbids any feature from importing another feature.**
The modular structure chosen for repo-extractability is the same structure that lets two people work
without colliding. Auth and storefront are disjoint by construction.

What is *not* parallelizable is the foundation — you cannot build features on a repo that does not
exist. But a second seam covers that window: **the contract and mock API layer are plain TypeScript
with no React Native dependency**, so they can be built from hour zero, in parallel with the scaffold.

### Assignment

| STEP | Owner | When | Scope |
|------|-------|------|-------|
| **Foundation** | Dev A — **Raul Angel** | Sat | RN bare scaffold (iOS + Android), TypeScript, module structure, **theme tokens both modes**, store + middleware wiring, navigation shell, and the **shared atoms both features need** (typography, pill button in both polarities, text field with error state, chip, icon wrapper, layout primitives) |
| **Contract & mock API** | Dev B — **Andres Montoya** | Sat | TypeScript interfaces, enums, response envelope, error shape; the written contract document; the mock client (Promises + artificial latency + failure injection); fixtures wired to `assets/placeholder-art/`; tests. **Zero RN dependency** — plain TS |
| **Auth feature** | Dev A — **Raul Angel** | Sun–Mon | Welcome, credentials, inline error state; auth slice + middleware; password field with visibility toggle; tests |
| **Storefront feature** | Dev B — **Andres Montoya** | Sun–Mon | Header, tab bar, **config-driven carousel component**, 2 variants, RTK Query feed endpoints, progress-bar and play-overlay molecules; tests |

**Why this pairing:** Dev B authors the contract, so Dev B owns the surface that consumes the most
data. Dev A authors the theme, so Dev A owns auth — the **light** surface mode, which is where token
gaps surface first, and the smallest screen set on which to find them.

### Synchronization — only two hard points

1. **Sat end of day** — foundation and contract both merge. Everything before this is independent.
2. **Tue AM** — integration, theme-swap verification, smoke on both platforms.

Between them, Sunday and Monday need **no coordination**: separate branches, disjoint directories,
independent test suites. Each dev works at their own pace.

### Collision hazards and their mitigations

| Hazard | Mitigation |
|--------|------------|
| Both devs need a shared atom the foundation didn't build | Whoever needs it first builds it in `shared/` and says so; the other imports. Rare if session 1.7 produces the atomic inventory *before* the foundation STEP — another reason 1.7 runs today |
| Merge conflicts in shared registration files (root reducer, navigation config, theme index) | The foundation STEP creates these with **both features' slots pre-registered as stubs**, so each dev edits only their own line |
| Foundation slips past Sat midday (RISK-0003) | **Dev B stops contract work and pairs on the scaffold.** The scaffold is the single point of failure for the whole schedule; two people on it beats one person on it and one person ahead of a blocked plan |

### Scope recovered from 1b

With two devs, the **two-step email → password auth split** returns to Phase 1a (Dev A). The
**live carousel variant stays in 1b** — later sessions (docs 04/07/11/13) parked `'live'` out of
the Phase-1 data model; this review keeps that (1.14). **UI tests stay in 1b**.

### STEP numbering

These four STEPs are **authored by the planning session** (`templates/planning-session.md`) on
Saturday morning, once STEP-1 closes — not invented here. Reserve each number on `prompts/`'s shared
trunk and push before branching, per `runbooks/collaboration.md`.

## 10. Placeholder brand

The demo ships a fictional brand, **QC+** / **MiQC+**, decided 2026-08-14 (OQ-08) and rebranded in
STEP-6. Assets live in `architecture/assets/brand/` (wordmarks light and dark, compact mark,
sub-brand strip) and mock artwork in `architecture/assets/placeholder-art/` at 2:3, 16:9, and 3:4 —
resolving OQ-09. See `architecture/assets/README.md`.

Two things carried forward:

- **Trademark exposure.** Generic fictional marks only (DF10); no third-party trademark art in shipped UI (RISK-0005).
- **Wordmarks outlined (OQ-13 closed, STEP-2.4).** Runtime copies in
  `quasar-qc-plus-mobile-app` `src/shared/assets/brand/` have `<text>` converted to paths so
  letterforms match on iOS and Android. Hub originals in `architecture/assets/brand/` remain
  provenance when present.

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Phase 1 milestone type | Functional POC / stakeholder demo, **with visual fidelity as an explicit goal** | Reference screenshots raised fidelity from implicit to a sign-off criterion | Not an MVP; no end-user value claimed |
| 2 | Phase 1 scope | Full two-step auth + home row-stack with **2 carousel variants in 1a** (CW + standard portrait); live + landscape in 1b; hero **chrome** Phase 2 (3:4 stand-in in 1a) | Covers every architectural pattern without the most expensive chrome | Demo shows less breadth than the reference home |
| 3 | Phase 1 split | **1a** (demo-gated, 2026-08-18) / **1b** (remainder, undated) | ~3.5 available days against a ~6–7 day scope; a polished subset beats a half-finished whole for a craft demo | Phase 1 is not complete on the sign-off date |
| 4 | Details screen | Phase 2; Phase 1a shows an alert with the title | Surface, not architecture — the alert proves the interaction path | Stakeholders see no drill-down on 18 Aug |
| 5 | Backend | **We build none.** We author the contract; mocks simulate real fetches | Backend team owns delivery; the contract makes Phase 3 a swap, not a rewrite | Phase 3 is externally gated |
| 6 | Contract ownership | Ours — session 1.11 promoted back into Phase 1a | The contract now *defines* the mock shapes, so it can't be deferred | Backend team inherits a contract they did not draft |
| 7 | Fetch simulation | Promises with artificial latency, failable on demand | Loading and error states are only real if the call can actually fail | Slightly slower mock layer than local data |
| 8 | Tests in the gate | Every reducer, RTK Query endpoint path, hook, and API layer tested; **UI tests deferred to 1b** | Logic tests carry the "this architecture is testable" argument at the lowest cost | No rendering-regression safety net on 18 Aug |
| 9 | Theme structure | Two surface modes (dark app / light auth) in one token set | The reference auth flow is light-on-white; one palette with exceptions would break criterion A1 | More token work up front |
| 10 | Architecture sessions | All core + included conditionals **Done** (1.14). 1.5/1.6a/1.8–1.10 ran after originally being Deferred | Only sessions that block code were meant to fit the window; the deferred set subsequently ran | — |
| 11 | Weekend work | Sat 15 + Sun 16 included | Sign-off date is not negotiable | No buffer for the scaffold risk |
| 12 | Trademark posture | Layout reproduced; all marks and key art substituted | Disney IP is not ours to ship | Demo looks like Disney+ in structure, not in branding |
| 13 | Team | **Two senior devs**, 15–18 Aug | Confirmed after the split was drafted for one | Foundation is still serial — a second dev does not halve the critical path |
| 14 | Parallelization seam | Four disjoint STEPs: foundation / contract+mocks / auth / storefront | DF5 (no cross-feature imports) already makes auth and storefront disjoint; the contract layer has no RN dependency so it parallelizes the scaffold window | Two hard sync points (Sat EOD, Tue AM); shared atoms need an owner rule |
| 15 | Scope recovered with the second dev | **Two-step auth in 1a**; live carousel **stays 1b**; **UI tests stay in 1b** | Two-step matches docs 03/16; `'live'` is out of the Phase-1 data model (doc 04) | Live badge on 18 Aug |
| 16 | Placeholder brand | Fictional **QC+** brand + abstract placeholder key art, authored in-repo | Unblocks the build without third-party assets; resolves OQ-08 and OQ-09 | Internal POC placeholder only (RISK-0005) |
| 17 | Connectivity | Online-only + shell NetInfo gate (doc 15) | Matches the no-internet reference; no offline cache in a mock demo | Offline browse; per-feature offline screens |
| 18 | Storefront pagination | Feature hooks + paginated mocks from day one | Organized loadMore; contract can page later | One-shot full-catalog fixture |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-07~~ | ~~Is a second senior dev available for 15–18 Aug?~~ **Resolved 2026-08-14: yes, two seniors.** See §9 | — | closed |
| ~~OQ-08~~ | ~~Placeholder branding — existing wordmark, or create one?~~ **Resolved 2026-08-14:** fictional **QC+** brand authored in `assets/brand/` (rebranded STEP-6) | — | closed |
| ~~OQ-09~~ | ~~Source of placeholder key art at 2:3, 16:9, 3:4~~ **Resolved 2026-08-14:** SVG art in `assets/placeholder-art/` | — | closed |
| ~~OQ-10~~ | ~~Does the backend team accept a contract they did not draft, and who reviews it?~~ **Superseded (1.11):** the contract now exists (`architecture/11-interface-contracts.md`); the question becomes **OQ-34**, and doc 11 §14 lists the eight items to settle with them | — | closed |
| OQ-11 | Who attends the 18 Aug sign-off, and what constitutes "passed"? | Stakeholders | P1 launch criterion |
| ~~OQ-12~~ | ~~Which dev is A and which is B?~~ **Resolved (planning session):** Dev A = **Raul Angel** (STEP-2, STEP-4, STEP-6); Dev B = **Andres Montoya** (STEP-3, STEP-5, STEP-7) | — | closed |
| ~~OQ-13~~ | ~~Who outlines the wordmark text before 18 Aug?~~ **Resolved (STEP-2.4):** outlined SVG paths shipped in app `src/shared/assets/brand/` | — | closed |
| ~~OQ-19~~ | ~~Pagination wire format (cursor vs offset) and first-page sizes~~ **Resolved (1.4):** opaque `nextCursor`; HomeFeed first page = hero + 15; CW is a separate endpoint. Envelope JSON names → 1.11 (OQ-22); tile page size → OQ-23 | — | closed |

Carried forward from doc 01 and still open: OQ-03 (production JWT claims / IdP vendor →
1.11 / Phase 3; mock claims closed in 1.6a), OQ-05 (Bitrise setup → Phase 2), OQ-06 (budget). **OQ-01** (final login/storefront UI) is
now **resolved** by `inputs/ui/streaming-reference-screens.md`. **OQ-04** (real streaming app
migration timeline) is unchanged and unblocking. **OQ-16** (persist library) is **resolved** by
1.3a: `react-native-encrypted-storage`. **OQ-02** (card schema) is **resolved** by 1.4.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-14 | STEP-1.2 | Initial draft from architecture session |
| v0.2.0 | 2026-08-14 | STEP-1.2 | Second senior dev confirmed → added §9 two-developer split (four disjoint STEPs, two sync points, collision mitigations) and recovered two 1b items into 1a. Added §10 placeholder brand (**QC+** + placeholder art). Closed OQ-07/08/09; opened OQ-12/13. Schedule table updated for parallel work. |
| v0.2.1 | 2026-08-16 | STEP-1.3 | DF1: axios (not fetch) inside the API module only. DF3: Keychain/Keystore persist of the auth slice from day one. DF5: shell is the composition root. |
| v0.3.0 | 2026-08-16 | STEP-1.3a | DF3 names `react-native-encrypted-storage`. Added DF11 (connectivity gate) and DF12 (storefront pagination). |
| v0.3.1 | 2026-08-16 | STEP-1.4 | DF12: opaque `nextCursor` (doc 04). Hero is in the Phase-1 feed contract/composition (ADR-0006); full chrome still OQ-24. Closed OQ-19. |
| v0.3.2 | 2026-08-17 | STEP-1.5 | DF6/DF12: Container variants include `progress`; both feeds are `Container[]` + `resources` (ADR-0007). |
| v0.3.3 | 2026-08-17 | STEP-1.6a | §7 amendment: 1.5 and 1.6a ran. Phase 3 JWT/refresh per doc 16. OQ-03 → 1.11 / Phase 3. |
| v0.3.4 | 2026-08-17 | STEP-1.7 | §2 "inert" tabs defined as tappable `ComingSoon` placeholders (doc 07 §4). Design system delivered; DF4/DF6/DF8 now have concrete token, component, and i18n specs. |
| v0.3.5 | 2026-08-17 | STEP-1.8 | §7 amendment: 1.8 ran (doc 08); 1.9 and 1.10 remain. §8 Tue-AM sync point now includes the release build + two-device install. DF11 notes ADR-0014 (interface-up connectivity). |
| v0.3.6 | 2026-08-17 | STEP-1.10 | §7 amendment: 1.9 (doc 09) and 1.10 (doc 10) ran — no originally-deferred session remains in the STEP-1 queue. 1.10 declines the telemetry stack and adds a shell-root error boundary (RISK-0014). No schedule change. |
| v0.3.7 | 2026-08-17 | STEP-1.11 | DF12 page sizes locked from doc 11 §6.3. **OQ-10 closed** — the contract exists; the backend-acceptance question becomes OQ-34 against doc 11 §14's checklist. No schedule change. |
| v0.3.8 | 2026-08-17 | STEP-1.14 | DF1/A3: I/O is RTK Query `baseApi` + axios interceptors; Phase 1 mocks are `axios-mock-adapter` on the same instance (ADR-0020). Goal sentence names Emotion + RTK Query (ADR-0019). |
| v0.4.0 | 2026-08-17 | STEP-1.14 | Cross-cutting: two-step auth in 1a; live stays 1b; hero 3:4 stand-in (OQ-24 closed); Decision 10/15 reconciled. STEP-1 sessions complete. |
| v0.4.1 | 2026-08-17 | planning session | Closed OQ-12: Dev A = Raul Angel, Dev B = Andres Montoya. §9 assignment table named. |
| v0.4.2 | 2026-08-18 | STEP-2.4 | Closed OQ-13: brand wordmark/strip SVG text outlined to paths in the app repo. §10 updated. |
