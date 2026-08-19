# quasar-qc-plus-mobile — Project Overview

<!-- PROJECT-STATUS: kickoff-complete -->
<!-- ^ Kickoff gate (do not delete this line). `init.sh` seeds it as "not-started". The agent
     flips it to "kickoff-complete" at the end of the bootstrap (BOOTSTRAP-PROMPT.md). While it
     reads "not-started", opening this project in an AI agent starts the kickoff interview
     automatically; once "kickoff-complete", agents resume from prompts/STEP-index.md instead. -->

<!-- CHECK-IN-CADENCE: 20 -->
<!-- ^ Check-in cadence (optional): aim for a Check-in STEP about every this-many STEPs. 20 is the
     recommended default — edit it for a tighter (e.g. 15) or looser (e.g. 50) rhythm, or delete the
     line to accept 20. It stays a judgment-based guideline. See METHOD.md §5 for how status.sh uses
     it (a heads-up 5 STEPs before the target, overdue 5 after). -->

## In one sentence

A React Native stakeholder demo of a Disney+-style streaming app — login plus a carousel storefront — built with Throughstone architecture so the team's real streaming product can adopt the same methodology later.

## The problem

The team needs a credible, functional reference app that shows stakeholders how a streaming product is shaped using the Throughstone template and disciplined architecture — not a throwaway prototype. Today there is no shared mobile codebase on this method; this project establishes the pattern (feature-based Clean Architecture, Redux Toolkit + RTK Query, Emotion theme, swappable API boundary) before migrating the production streaming app.

## Who it's for

- **Primary:** Internal stakeholders evaluating the Throughstone approach and mobile architecture.
- **Secondary:** The development team as a migration template for the real streaming app.
- **Not in v1:** End-user customers at scale; content operators; compliance reviewers (no regulated flows yet).

## What it does (core capabilities)

- **Login** — two-step in Phase 1a: Welcome → email → password, with an inline credentials error (F2). UI locked from `inputs/ui/streaming-reference-screens.md`.
- **Storefront** — home/browse with **2 carousel variants in 1a** (continue-watching + standard portrait) plus pack **hero banner chrome**; live + landscape in 1b; neighbor-peek spotlight and filter rail remain Phase 2. Card schema is `architecture/04-data-model.md`.
- **Data layer** — Redux Toolkit store with **RTK Query `baseApi`**; axios interceptors attach JWT; mock content via `axios-mock-adapter` on the same instance until a real backend exists.
- **Auth boundary** — hardcoded demo credentials in v1; code structured so swapping to a real API + JWT is a localized change.

## What it does NOT do (for now)

**Not yet (deferred — target is a full streaming app eventually):**

- Video player / playback
- Settings screen
- Search
- Profiles (multi-user household)
- Offline downloads
- Parental controls

**Never:** *(none — long-term goal is full streaming-app feature parity; v1 is intentionally narrow.)*

## Scale & shape

- **Launch:** Stakeholder demo — internal/small audience, both platforms shown side by side.
- **Shape:** Cross-platform **mobile app** (React Native) on **iOS and Android**; no real backend in v1 — dummy API + frontend mocks.
- **Year-one (aspirational):** Same architecture extended to production streaming features and a real backend/API.

## Release stage / launch target

Internal pre-launch demo for stakeholders — functional on iOS and Android simulators/devices, not a public store release.

## Constraints & must-haves

- **React Native** — single codebase, iOS + Android.
- **Redux Toolkit** — global state; **RTK Query** (`baseApi` + feature `injectEndpoints`) for APIs and token lifecycle; axios interceptors on the shared instance attach `Authorization`.
- **Emotion** — `@emotion/native` styled components + `@emotion/react` `ThemeProvider` with a token palette (color, type, space).
- **TypeScript** — interfaces, enums, and typed values throughout.
- **Atomic design** — atoms → molecules → organisms for reusable UI.
- **Modular file system** — feature modules colocate related code (screens, components, helpers, state, API adapters) under `src/features/<name>/` using the auth reference layout ([`architecture/03-architecture-overview.md`](architecture/03-architecture-overview.md) §8.1.1); organized for potential extraction into separate repositories.
- **No real backend in v1** — dummy API + mocked responses; naming and boundaries must anticipate production endpoints.

## Sensitive data & risk

- v1 uses **fake demo credentials only** (no real PII).
- Auth flow must be designed so migrating to **JWT** from a real API is straightforward — token storage, RTK Query `baseApi`, and axios interceptors should not assume hardcoded login long-term.
- No payments, health data, or regulated content in v1.

## Known unknowns

- Production backend contract acceptance and JWT claims shape for the real IdP (**OQ-34**, **OQ-03**) — dummy API stands in.
- Real streaming app migration timeline and which modules move first (**OQ-04**).
- Who is Dev A / Dev B, the application repo name, and who owns the sign-off binary (**OQ-12**, **OQ-18**, **OQ-28**) — **closed by the planning session:** Dev A = Raul Angel, Dev B = Andres Montoya; app repo = `quasar-qc-plus-mobile-app` (renamed STEP-6.1); sign-off binary owner = Raul Angel (STEP-6). Two demo devices are named at the STEP-6 pre-flight.

## Anything else

- **Inspirations:** Disney+ (UX patterns — login, horizontal carousels, varied tile sizes).
- **Strategic intent:** This repo is a **methodology showcase** first; production feature parity comes in later phases after the team validates the Throughstone workflow on mobile.
- **Design inputs:** Login and storefront UI locked from `inputs/ui/streaming-reference-screens.md`.
