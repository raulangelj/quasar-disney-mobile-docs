# Doc 01 — System Overview, Requirements & Non-Goals

**Version:** v0.1.4
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.6a)
**Audience:** Product stakeholders, mobile developers, backend team, QA

> What quasar-disney-mobile is building in v1, who it serves, how success is measured, and what is deliberately out of scope.

## 1. Problem & Value

The team needs a credible, functional reference app that shows stakeholders how a streaming product is shaped using the Throughstone template and disciplined architecture — not a throwaway prototype. Today there is no shared mobile codebase on this method; this project establishes the pattern before migrating the production streaming app: **atomic-design UI components**, **Styled Components** for styling, **custom hooks** to separate business logic from presentation, a **Redux store with middleware** routing API calls through a swappable boundary, **thoughtful TypeScript interfaces, enums, and types** across the data and UI layers, and a **modular file structure** ready for real backend integration.

**Why now:** Internal stakeholders need a working demo to validate the Throughstone mobile methodology before the real streaming app migrates to the same patterns.

## 2. Users & Stakeholders

| Group | Role | What they need from this |
|-------|------|--------------------------|
| **Primary** | Internal stakeholders | Evaluate the Throughstone approach and mobile architecture via a working demo |
| **Secondary** | Development team | A migration template for the real streaming app |
| **QA** | Build validation | Review generated **iOS and Android builds** (likely via **Bitrise** CI once wired) — stable, testable artifacts on both platforms |
| **Backend team** | API ownership | A clear **API boundary** and eventual backend repo they own; mobile app structured so swapping dummy endpoints for real APIs is localized |
| **Out of v1** | End-user customers at scale, content operators, compliance reviewers | Not targeted yet — no regulated flows |

## 3. Success Criteria

| Criterion | Measurable target |
|-----------|-------------------|
| **Demo-ready on both platforms** | App runs on **iOS and Android** (simulator or device) with login → storefront flow completable end-to-end |
| **Stakeholder sign-off** | Internal stakeholders can walk through the demo and confirm the **architecture patterns** (atomic design, hooks, Redux middleware, typed models, swappable API, theme tokens) are visible and credible |
| **QA-validated builds** | QA receives **installable builds** for both platforms (Bitrise once CI exists) and can execute a basic smoke checklist without blockers |
| **API swap readiness** | Replacing dummy auth/content endpoints with real backend URLs requires changes **only in the API/middleware layer** — no storefront or hook rewrites |
| **Theme swap readiness** | A **central theme** (color variables, typography tokens, spacing) lives in one place; re-skinning for a different product means **updating theme tokens only** — not hunting colors/styles across screens |

## 4. Scope

| Core (in) | Not now (deferred) | Not ever |
|-----------|-------------------|----------|
| Login screen (email/password; UI from reference designs when available) | Video player / playback | *(none — long-term goal is full streaming-app feature parity)* |
| Storefront / home with **up to 8 carousel variants** (aspect-ratio-driven tile sizes); **paginated** rows and tiles via storefront hooks | **Content details screen** (metadata, description, cast, "similar to this" row) | |
| **Tap behavior v1:** Alert showing content title on card tap | Settings screen | |
| Central **theme** (tokens consumed by Styled Components; no hardcoded colors) | Search | |
| Atomic-design UI (atoms → molecules → organisms) | Multi-user profiles (household) | |
| Custom hooks separating business logic from presentation | Offline downloads | |
| Redux store + middleware for async/API calls | Parental controls | |
| Typed data layer (interfaces, enums, types) | Real backend / production API | |
| Dummy API + frontend mocks | Real JWT auth (structure ready; demo credentials in v1) | |
| Demo auth (hardcoded credentials; JWT-ready structure) | Bitrise CI / automated builds (QA builds follow scaffold) | |
| Cross-platform mobile (bare **React Native**, iOS + Android — **no Expo**) | Public app store release | |
| Modular feature-oriented file structure | Payments / subscriptions | |
| Analytics hook **design** with console.log stubs in v1 | Full analytics integration | |

## 5. Constraints

| Category | Constraint |
|----------|------------|
| **Timeline** | ~~**3–4 days** for v1 (login + storefront)~~ — **superseded by doc 02 §8.** Stakeholder sign-off is fixed at **2026-08-18**; Phase 1 is split into 1a (demo-gated) and 1b. The 3–4 day figure assumed a single login screen, before the reference screenshots expanded the auth flow to three screens plus an error state. |
| **Team** | **1 senior dev** (owner); **+1 senior** if needed; later **1–2 senior or mid-level** devs |
| **Budget** | **TBD** — not a blocker; management decides while dev proceeds |
| **Tech stack** | Bare **React Native** (no Expo), iOS + Android, **TypeScript**, **Redux + middleware**, **Styled Components**, **atomic design**, **custom hooks** |
| **Architecture** | Modular feature structure; swappable API boundary; central theme tokens; typed models throughout |
| **Backend** | **No real backend in v1** — dummy API + frontend mocks; backend team owns separate repo later |
| **Auth** | Demo credentials only in v1; JWT-ready structure required |
| **Integrations (v1)** | Endpoints only — no SSO, payments, or third-party services |
| **Analytics (deferred)** | Dedicated analytics hook later; v1 uses **console.log stubs** only |
| **Data / compliance** | No real PII, payments, health data, or regulated content in v1 |
| **Release target** | Internal stakeholder demo — simulators/devices, **not** public app store |
| **Design** | UI driven by stakeholder reference images (login, storefront); assets TBD in `inputs/` |
| **CI / QA** | Bitrise builds for iOS/Android expected later; not a v1 blocker but QA is a stakeholder |
| **Work split (rough)** | ~~~1 day login + theme/scaffold; ~2–3 days storefront (8 carousel variants + mocks)~~ — **superseded by doc 02 §8.** Config-driven carousel still holds; variant count settled at **5** (2 in Phase 1a). |

## 6. Assumptions

| # | Assumption | If wrong… |
|---|------------|-----------|
| 1 | **Two screens (login + storefront) fit in 3–4 days** with one senior dev | Scope or timeline slips; need second senior or cut carousel variants |
| 2 | **Bare React Native** (no Expo) is acceptable for iOS + Android | Revisit toolchain, native module setup, or CI complexity |
| 3 | **Dummy API + frontend mocks** are enough for the stakeholder demo | Need a real backend sooner than planned |
| 4 | **Stakeholder reference images** for login/storefront arrive in time (or dev can proceed with placeholders) | UI polish blocked; demo looks unfinished |
| 5 | **Hardcoded demo credentials** suffice for v1; no real user accounts | Confirmed in 1.6a (doc 16). Phase 3 buys an IdP behind our API |
| 6 | **No regulated data** in v1 — fake credentials only | Privacy/compliance session (1.7a) becomes mandatory earlier |
| 7 | **Backend team** will own a separate API repo and adopt the mobile app's contract later | Interface-contracts session (1.11) must lock shapes now |
| 8 | **Theme token swap** is sufficient for re-skinning future projects | May need multi-theme runtime switching or white-label build variants |
| 9 | **Carousel mock data** can live in the frontend until backend exists | Content schema decisions deferred but shouldn't block v1 |
| 10 | **Analytics** can wait — console.log stubs are fine for v1 | Product asks for funnel metrics before v1 demo |
| 11 | **Bitrise CI** can follow initial scaffold; manual builds OK for first QA pass | QA blocked without installable artifacts |
| 12 | **Disney+-style UX** (carousels, varied tile sizes) is the right demo narrative | Stakeholders want a different flow emphasis |
| 13 | **Eight carousel layout variants** are implemented as one **config-driven carousel component** (aspect-ratio presets); building them fits within **~1–1.5 days** of the 3–4 day timeline | More layouts needed → component explosion or timeline slip |
| 14 | **Alert-on-tap** is acceptable for v1 demo (no details navigation) | Stakeholders expect drill-down → details screen moves from deferred to must-have |

## 7. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **3–4 day timeline too tight** for bare RN scaffold + login + 8 carousels | Medium | High | Config-driven carousel; second senior dev on standby; cut carousel count before cutting architecture patterns |
| **Reference UI images delayed** | Medium | Medium | Ship with placeholder layout matching Disney+ patterns; refine when inputs arrive |
| **Backend contract undefined** | High | Medium | Lock interface shapes in session 1.11 early; dummy API mirrors expected production shapes |
| **QA blocked without Bitrise** | Medium | Medium | Manual dev builds for first smoke pass; prioritize CI in infrastructure session (1.8) |
| **Scope creep** (details screen, playback, real auth) before demo | Medium | High | Non-goals table is the guardrail; deferrals require explicit STEP, not ad-hoc additions |
| **Theme abstraction insufficient** for future white-label | Low | Medium | Session 1.7 (UI / Design System) validates token coverage |

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Problem statement | Throughstone methodology showcase + migration template | Stakeholders need proof of patterns, not a throwaway UI | v1 optimizes for teachability over feature completeness |
| 2 | Stakeholders | Stakeholders, dev team, QA, backend team | Each group consumes different outputs (demo, template, builds, API contract) | End users and compliance not in v1 audience |
| 3 | Success criteria | Demo on both platforms + pattern sign-off + API/theme swap readiness | Measurable without vanity metrics | No performance/load targets in v1 |
| 4 | Core capabilities | Login + storefront (8 carousels) + full architecture stack | Minimum credible streaming-app slice | Details, playback, search deferred |
| 5 | Non-goals | Playback, details, settings, search, profiles, offline, real backend/auth, store release | Protect 3–4 day timeline | Long-term parity still the strategic goal |
| 6 | Constraints | 3–4 days, 1 senior dev, bare RN, no Expo, budget TBD | Known team and timeline reality | Expo-managed workflow ruled out |
| 7 | Assumptions | Config-driven carousels, alert-on-tap, dummy API, theme tokens | De-risk timeline | Custom carousel per row would blow budget |
| 8 | Key risks | Timeline, missing designs, undefined backend contract | Early visibility for phasing session | |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-01 | Final login and storefront UI from stakeholder reference images | Stakeholders / design | 1.7 UI / Design System; `inputs/` |
| ~~OQ-02~~ | ~~Exact carousel card metadata schema~~ **Resolved (1.4):** Title fields + artwork map + CW progress fields in `architecture/04-data-model.md`. JSON names → 1.11 | — | closed |
| OQ-03 | Production backend contract and JWT claims shape | Backend team | 1.11 Interface Contracts; Phase 3 |
| OQ-04 | Real streaming app migration timeline and which modules move first | Product / eng leadership | 1.2 Phasing & Roadmap |
| OQ-05 | Bitrise project setup and signing credentials | DevOps / mobile | 1.8 Infrastructure & Deployment |
| OQ-06 | Budget and tooling approvals | Management | 1.8 Infrastructure |

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-14 | STEP-1.1 | Initial draft from architecture session |
| v0.1.1 | 2026-08-14 | STEP-1.2 | §5 timeline and work-split constraints superseded by doc 02 (fixed 2026-08-18 sign-off; Phase 1 split into 1a/1b). OQ-01 resolved by `inputs/ui/disney-plus-reference-screens.md`. |
| v0.1.2 | 2026-08-16 | STEP-1.3a | Storefront scope: paginated feed/carousels via feature hooks (doc 15 / ADR-0005). |
| v0.1.3 | 2026-08-16 | STEP-1.4 | Closed OQ-02 (Title / CW / artwork schema in doc 04). |
| v0.1.4 | 2026-08-17 | STEP-1.6a | Assumption 5 confirmed (demo credentials). OQ-03 now 1.11 / Phase 3; mock claims closed in doc 16. |
