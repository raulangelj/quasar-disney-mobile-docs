# Doc 03 — Architecture Overview & Component Boundaries

**Version:** v0.3.2
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.6a)
**Audience:** Mobile developers, backend team, QA

> How quasar-disney-mobile is cut into components, how those pieces talk, and which boundary is the only one that needs a formal contract.

## Table of Contents

1. [Client surfaces](#1-client-surfaces)
2. [Architecture style](#2-architecture-style)
3. [Component diagram](#3-component-diagram)
4. [Component table](#4-component-table)
5. [Boundaries & contract candidates](#5-boundaries--contract-candidates)
6. [Key flows](#6-key-flows)
7. [Build vs. buy](#7-build-vs-buy)
8. [Import rules](#8-import-rules)

---

## 1. Client surfaces

**One surface:** a **React Native mobile app on iOS and Android**. No web, desktop, CLI, or public API-as-a-product.

| Gate | Call |
|------|------|
| **UI / Design System (1.7)** | **Include** — styled screens are the demo |
| **Native-app session (1.3a)** | **Include** — already slotted; covers offline, permissions, pinning, distribution |

A companion web/admin surface or treating the API as a standalone product can land later without rewriting the mobile boundary.

## 2. Architecture style

**Modular monolith** — one React Native app, five in-process modules, hard import rules. The **app shell is the composition root**: it is the only module allowed to import features in order to register screens, reducers, and middleware.

This is the simplest shape that still makes auth and storefront extractable later (doc 02 DF5). It **forecloses** splitting into services, a separate mock HTTP server process, or extracting npm packages in Phase 1.

Internal partition of Shared is folders, not packages: `shared/theme/`, `shared/ui/`, `shared/i18n/`, `shared/analytics/`. There is **no types module** — wire types live in the API module, feature types live with the owning feature, component props live next to the component.

Code lives in a **single application repo**, named when the foundation STEP creates it. No backend repo is created by this project.

## 3. Component diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         App shell                               │
│   RN host · navigation · store · persist rehydrate · NetInfo    │
│                                                                 │
│         ┌──────────────┐         ┌──────────────────┐           │
│         │ Auth feature │         │ Storefront       │           │
│         │              │         │ feature          │           │
│         └──────┬───────┘         └────────┬─────────┘           │
│                │                          │                     │
│                │     (middleware only)    │                     │
│                └────────────┬─────────────┘                     │
│                             ▼                                   │
│                    ┌─────────────────┐                          │
│                    │   API module    │                          │
│                    │ axios + mocks   │                          │
│                    └─────────────────┘                          │
│                             ▲                                   │
│         ┌───────────────────┴───────────────────┐               │
│         │           Shared kernel               │               │
│         │  theme / ui / i18n / analytics        │               │
│         └───────────────────────────────────────┘               │
│                                                                 │
│   Auth ↛ Storefront. Shell reads auth selectors to switch nav.  │
└─────────────────────────────────────────────────────────────────┘
```

A future HTTP backend is **not a component of this system**. It is a Phase-3 swap behind the API module.

## 4. Component table

| Name | Responsibility | Tech |
|------|----------------|------|
| **App shell** | Boots the RN host on iOS/Android: navigation, store/middleware registration, theme provider, persist rehydrate, **cold-start loading** until `/me` + HomeFeed + ContinueWatching complete, **NetInfo connectivity overlay** | Bare RN, TypeScript, React Navigation, Redux Toolkit store composition, `@react-native-community/netinfo` |
| **Auth feature** | Welcome → email → password → session. Owns the **auth slice** (JWT) and the **user slice** (`userName` from `/me`). Imports `shared/` only | RTK slices, custom hooks, Styled Components |
| **Storefront feature** | Home/browse and config-driven carousels. Owns the content slice (not persisted), **pagination hooks**, **client merge** of hero + `progress` (CW) + other containers, **silent CW reload**. Imports `shared/` only | RTK slice, custom hooks, Styled Components |
| **Shared kernel** | Theme (both surface modes), atomic UI, i18n string tables, analytics hook stub | Styled Components + tokens; folders `theme/`, `ui/`, `i18n/`, `analytics/` |
| **API module** | Client + mock adapter. The only I/O boundary. Single **axios** instance (base URL, auth-header interceptor, error mapping). Mocks return Promises with latency and injectable failure | TypeScript types, axios, mock adapter |

## 5. Boundaries & contract candidates

Most handoffs are **in-process imports**, not network APIs. The only contract worth writing now is the **API module’s request/response shape** — mocks today, real HTTP later. Final contract policy, source of truth, and artifact locations land in session 1.11.

| Boundary | What crosses it | Sync / async | Data owner | Likely contract style | Notes for 1.11 |
|----------|-----------------|--------------|------------|----------------------|----------------|
| Shell → Auth, Storefront, Shared, API | Screen / reducer / middleware registration | Sync | Shell owns composition | None (in-process public exports) | Do not OpenAPI the shell |
| Auth → Shared · Storefront → Shared | Atoms, tokens, i18n, analytics stub | Sync | Shared kernel | None | Public component/token API only |
| Auth ↛ Storefront | Nothing | — | — | — | Shell reads auth selectors to switch navigators |
| Auth / Storefront → API module (via middleware only) | Login, **`/me`**, **paginated HomeFeed** (`Container[]`, hero + 15), **paginated ContinueWatching** (`Container[]`, `variant: "progress"`) | Async (Promises; mock latency/failure) | API module owns wire shapes; features own slice state | **Yes** — TS interfaces, enums, envelope, error shape, **`nextCursor`**, **`resources: Card[]`** | Transport is **axios**, not `fetch`. Screens/hooks never import `axios`. JSON names in 1.11 (OQ-22). JWT on `/me` and both feeds |
| API module → future HTTP backend | Same payloads over the network | Async | Backend team later (not built here) | Same shapes; transport TBD in 1.11 | Not a component we build; do not invent a mock server just to have HTTP |

## 6. Key flows

### Flow 1 — Sign-in (success, inline error, and restore)

1. App shell boots. **redux-persist** rehydrates the **auth slice only** from **`react-native-encrypted-storage`** (Keychain / EncryptedSharedPreferences). The user slice is empty until `/me`.
2. If NetInfo reports no network, the shell shows the **no-internet overlay** (doc 15) on top of whichever navigator the session implies; it does not unmount it. Reconnect or `REINTENTAR` hides the overlay and resumes auth vs storefront from auth state.
3. If a session exists, shell shows the **cold-start loading screen**, then `/me` + HomeFeed + ContinueWatching (doc 04). Success → app navigator, **dark** theme, composed home. `/me` 401 or expired mock JWT → clear session → Welcome.
4. If not, shell applies the **light** theme and mounts the unauthenticated navigator.
5. Auth shows Welcome (Shared atoms). CTA → email → password (two-step, Phase 1a).
6. Submit: Auth dispatches → middleware → **API module** (axios instance or mock). No screen imports `axios`.
7. **Success:** mock returns a JWT (7-day `exp`) → auth slice stores it → persist writes it → **`/me` hydrates the user slice** → same three-call gate as cold start → shell switches to the app navigator and **dark** theme.
8. **Failure:** mock fails on demand → same error shape as the future API → credentials screen renders the reference inline error (red underline + message). Not an alert, not a local `if (password !== …)` branch. Not the no-internet overlay.

### Flow 2 — Storefront load and card tap

1. Shell mounts Storefront as the home tab after the boot gate (other tabs inert).
2. Storefront **composes** `[hero container] + [progress container] + [other HomeFeed containers]` from the two feed responses (ADR-0006 / ADR-0007). Pagination hooks dispatch `loadMore` → middleware → API module. End-reached calls `loadMore`.
3. When the storefront screen is shown again, Storefront **silently refetches ContinueWatching** and replaces that **`progress` container** only.
4. Card tap goes through the **same handler** that will later navigate → Alert with the title (Phase 1). Tile components do not know about navigation.

## 7. Build vs. buy

Phase 1 is a mobile demo with mocks. Almost everything is built in-app; we only take OSS libraries, not hosted products.

| Capability | Build / buy | Notes |
|------------|-------------|--------|
| Auth (demo credentials, session slice) | **Build** (Phase 1 mock) | No Auth0 / Firebase / Cognito SDK. Phase 3 **buys** a managed IdP **behind our API** (ADR-0009); a real JWT is still a swap at the API module |
| Content / home feed | **Build** (fixtures + mock adapter) | No CMS, no mock HTTP server |
| Storefront UI, theme, atoms | **Build** | Styled Components + tokens |
| HTTP + persistence + connectivity | **Buy (OSS)** | axios, Redux Toolkit, redux-persist, **`react-native-encrypted-storage`**, `@react-native-community/netinfo` |
| Navigation | **Buy (OSS)** | React Navigation |
| Payments, search, email, push, video CDN | **Neither** | Out of Phase 1 |
| Analytics | **Build stub** | Hook + `console.log`; no vendor |
| Backend / IdP / Bitrise | **Not now** | Backend is a future swap; CI is Phase 2 |

**Hard dependencies:** React Native, Redux Toolkit, React Navigation, Styled Components, axios, redux-persist, `react-native-encrypted-storage`, `@react-native-community/netinfo`. No cloud vendor lock-in in Phase 1.

## 8. Import rules

| From → To | Allowed? |
|-----------|----------|
| Shell → Auth, Storefront, Shared, API module | **Yes** — composition root |
| Auth → Shared · Storefront → Shared | **Yes** |
| Auth / Storefront → API module | **Only via middleware** — never `axios`/`fetch` from a screen or hook |
| Auth → Storefront · Storefront → Auth | **No** |
| Shared → `features/` | **No** |
| Persist | **Auth slice only**, into **`react-native-encrypted-storage`** — not AsyncStorage, not the rest of the store |

Redux: **Redux Toolkit** for slices plus **explicit async middleware** so the API boundary stays visible (not hidden behind `createAsyncThunk`).

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Client surfaces | RN iOS + Android only | Matches docs 01/02; demo is the mobile app | No web/admin/CLI/API-as-product in Phase 1 |
| 2 | Top-level components | Five in-app: shell, auth, storefront, shared kernel, API module | No backend component — we do not build a server | A Phase-1 backend service or mock HTTP process |
| 3 | Shared kernel grain | One component; folders `theme/`, `ui/`, `i18n/`, `analytics/` | Extraction path later without package ceremony under the deadline | npm workspaces / a global types module |
| 4 | Architecture style | Modular monolith; shell is composition root | DF5 extractability + two-dev split without services | Microservices; features importing each other |
| 5 | Contract candidate | API module wire shapes only | Only boundary that will cross a network later | Formal contracts between in-process modules |
| 6 | HTTP client | axios, API module only | Single instance, interceptors, swap-ready | `fetch`/`axios` from screens or hooks |
| 7 | Session persistence | redux-persist, auth slice, **`react-native-encrypted-storage`** (Keychain / EncryptedSharedPreferences) | Real JWT later is a payload change, not a storage rewrite (OQ-16 closed) | AsyncStorage for tokens; persisting the whole store; custom Keychain module |
| 8 | State libraries | RTK + explicit async middleware + React Navigation | Teaching pattern stays visible; stack already locked | Zustand/MobX, Expo Router, Expo |
| 9 | Build vs. buy | Build app + mocks; buy OSS libs only; no BaaS in Phase 1. Phase 3 IdP is buy-behind-API (ADR-0009) | Phase 1 has no real backend | Auth0/Firebase/CMS as Phase-1 dependencies |
| 10 | Connectivity | Shell-owned NetInfo overlay | One gate matching the reference; restore by auth state | Per-feature offline screens; last-known-home cache |
| 11 | Storefront paging | Feature hooks + paginated mocks | Organized loadMore; contract can page in Phase 3 | One-shot full-catalog payload |
| 12 | User profile | Memory-only user slice; `/me` with JWT on every cold start | Matches production; persist stays auth-only (ADR-0003) | Persisting `userName`; profile inside the auth slice |
| 13 | Home composition | Two GETs of **`Container[]`**; CW `progress` inserted under hero (ADR-0006 / ADR-0007) | Silent CW reload; real API split; one Container/Card component | Single home payload; distinct CW types |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-16~~ | ~~Persist backend library: `react-native-encrypted-storage` vs a thin `react-native-keychain` adapter~~ **Resolved (1.3a):** `react-native-encrypted-storage` | — | closed |
| OQ-17 | Mock strategy: axios-mock-adapter on the real instance vs a separate mock client behind the same functions | Mobile | 1.11 Interface Contracts; contract+mocks STEP |
| OQ-18 | Application repo name when created | Eng leadership | Planning session / foundation STEP |
| ~~OQ-19~~ | ~~Pagination wire format: cursor vs offset, envelope fields~~ **Resolved (1.4):** opaque `nextCursor`. JSON names → 1.11 (OQ-22) | — | closed |

Carried forward: OQ-10 (backend team accepts the contract → 1.11), OQ-12 (who is Dev A / Dev B). **OQ-02** is closed (doc 04).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-16 | STEP-1.3 | Initial draft from architecture session |
| v0.2.0 | 2026-08-16 | STEP-1.3a | Shell owns NetInfo connectivity overlay; storefront pagination hooks; persist engine locked to `react-native-encrypted-storage`. Closed OQ-16; opened OQ-19. |
| v0.3.0 | 2026-08-16 | STEP-1.4 | User slice + `/me`; two feed endpoints + boot loader; closed OQ-19. |
| v0.3.1 | 2026-08-17 | STEP-1.5 | HomeFeed/CW are `Container[]` + `resources: Card[]`; CW variant `progress` (ADR-0007). |
| v0.3.2 | 2026-08-17 | STEP-1.6a | Phase 3 IdP is buy-behind-API (ADR-0009); Phase 1 mock auth unchanged. |
