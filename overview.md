# quasar-disney-mobile — Project Overview

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

The team needs a credible, functional reference app that shows stakeholders how a streaming product is shaped using the Throughstone template and disciplined architecture — not a throwaway prototype. Today there is no shared mobile codebase on this method; this project establishes the pattern (modular structure, Redux data layer, swappable API boundary) before migrating the production streaming app.

## Who it's for

- **Primary:** Internal stakeholders evaluating the Throughstone approach and mobile architecture.
- **Secondary:** The development team as a migration template for the real streaming app.
- **Not in v1:** End-user customers at scale; content operators; compliance reviewers (no regulated flows yet).

## What it does (core capabilities)

- **Login screen** — simple email/password gate; UI to be refined from reference designs (see `inputs/` when provided).
- **Storefront** — home/browse experience with multiple carousel layouts (varying card sizes) displaying mocked content cards.
- **Data layer** — Redux store with middleware routing API calls to dummy endpoints; mock content served from the frontend until a real backend exists.
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
- **Redux** — global state with **middleware** for async/API side effects; API module named and structured for easy replacement with a real backend.
- **Styled Components** — styles separated per screen or component.
- **TypeScript** — interfaces, enums, and typed values throughout.
- **Atomic design** — atoms → molecules → organisms for reusable UI.
- **Modular file system** — feature modules colocate related code (UI, state, types, API adapters) and stay organized for potential extraction into separate repositories.
- **No real backend in v1** — dummy API + mocked responses; naming and boundaries must anticipate production endpoints.

## Sensitive data & risk

- v1 uses **fake demo credentials only** (no real PII).
- Auth flow must be designed so migrating to **JWT** from a real API is straightforward — token storage, middleware, and API client should not assume hardcoded login long-term.
- No payments, health data, or regulated content in v1.

## Known unknowns

- Final login and storefront UI from stakeholder reference images (pending in `inputs/`).
- Exact carousel variants and card metadata schema (to be locked in architecture sessions).
- Production backend contract and JWT claims shape (deferred — dummy API stands in).
- Whether Expo or bare React Native best fits team constraints (to decide in architecture).
- Real streaming app migration timeline and which modules move first.

## Anything else

- **Inspirations:** Disney+ (UX patterns — login, horizontal carousels, varied tile sizes).
- **Strategic intent:** This repo is a **methodology showcase** first; production feature parity comes in later phases after the team validates the Throughstone workflow on mobile.
- **Design inputs:** User will provide UI reference images for login (and likely storefront) — save to `inputs/` when received.
