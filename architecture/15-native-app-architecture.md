# Doc 15 — Native App Architecture

**Version:** v0.2.6
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.14)
**Audience:** Mobile developers, QA

> Device-side decisions for the React Native iOS + Android client: platform, connectivity, on-device storage, permissions, security posture, distribution, and performance — including how the storefront paginates.

## Table of Contents

1. [Platform strategy](#1-platform-strategy)
2. [Offline, sync, and the connectivity gate](#2-offline-sync-and-the-connectivity-gate)
3. [On-device storage & state](#3-on-device-storage--state)
4. [Push notifications](#4-push-notifications)
5. [Device permissions & capabilities](#5-device-permissions--capabilities)
6. [Device security](#6-device-security)
7. [Distribution & release](#7-distribution--release)
8. [Device performance & storefront pagination](#8-device-performance--storefront-pagination)

---

## 1. Platform strategy

**Bare React Native, iOS + Android, no Expo.** One TypeScript codebase, two native binaries. No web, desktop, PWA, Flutter, or Swift/Kotlin dual-native track.

This is the same surface locked in docs 01 and 03. This session confirms it rather than reopening it.

**Forecloses:** Expo (EAS, OTA, managed workflow), Electron/Tauri, a second native codebase.

Keychain / Keystore access is **only** through existing npm packages — no custom native modules for secure storage.

## 2. Offline, sync, and the connectivity gate

**Online-only.** No offline content cache, no download-and-watch, no write-sync, no conflict rules. Downloads remain Phase 4+. The catalog RTK Query cache is not persisted (doc 03).

Session restore from secure storage is **not** offline mode: it is a local auth token, not a content store.

### Connectivity gate

When the device has no usable network, the **app shell** shows a full-screen overlay matching `inputs/ui/07-no-internet.png` (transcribed in `inputs/ui/disney-plus-reference-screens.md` §7):

- Dark near-black surface (app-theme family, not the light auth sheet).
- Centered copy: *Es necesario revisar tu conexión a internet. Volveremos a cargar automáticamente la pantalla una vez que se establezca la conexión.*
- White pill CTA, black uppercase `REINTENTAR`.
- Copy goes through i18n (DF8); Spanish is the reference locale.

**Behavior**

1. `@react-native-community/netinfo` is the connectivity source. **"Usable network" means the
   interface is up** — plain `isConnected`, with **no internet-reachability probe** and no
   captive-portal detection (**ADR-0014**, resolving OQ-21). Phase 1 data is all in-process
   mocks, so the overlay guards no request; keying on reachability would let a captive-portal
   venue wifi block a fully working app at the 18 Aug sign-off. The accepted tradeoff —
   connected-but-not-reachable reports online — is harmless until a real host exists (OQ-29,
   Phase 3).
2. Overlay is **shell-owned**. Auth and Storefront do not implement their own offline screens.
3. Overlay sits **on top of** the current navigator; it does not unmount auth/storefront.
4. **Auto-restore:** when NetInfo reports connectivity, hide the overlay and resume from **auth state** — storefront if a session exists, auth flow if not. Refetch the active surface (storefront home if authenticated).
5. **`REINTENTAR`:** the same restore path, invoked manually (in case NetInfo is stale).
6. Cold start while offline: rehydrate the auth slice as usual, keep the overlay up until online, then choose auth vs storefront from the rehydrated session.

Mocks still simulate fetch failure (DF2). **No network** is a shell concern; **request failed** is a feature concern (inline error on credentials, storefront error/empty). Do not conflate them.

**Forecloses:** offline browse, last-known-home cache, sync engines in Phase 1.

## 3. On-device storage & state

| What | Where | Engine |
|------|-------|--------|
| Auth slice (JWT / `isAuthenticated` / `expiresAt`) | Keychain (iOS) / EncryptedSharedPreferences (Android) | **redux-persist** + **`react-native-encrypted-storage`** |
| User slice (`userName`) | Memory only | RTK; refill via `/me` on every cold start (doc 04) |
| Content / storefront slice | Memory only | RTK |
| UI flags, errors, pagination cursors | Memory only | RTK / hook state |
| Analytics | `console.log` stubs | none |

Screens never call the secure-store API. They read the auth slice after the shell finishes rehydrate (ADR-0003).

OQ-16 is **closed:** `react-native-encrypted-storage`, not a `react-native-keychain` adapter. It already exposes `getItem` / `setItem` for redux-persist on both platforms.

**Forecloses:** AsyncStorage for tokens, SQLite/MMKV in Phase 1, persisting the RTK Query cache, a custom Keychain native module.

## 4. Push notifications

**None in Phase 1.** No APNs, FCM, OneSignal, device-token plumbing, or notification permission prompt.

Revisit when a later phase actually sends notifications. Do not add Firebase “just in case.”

**Forecloses:** reminder / new-content pushes on the 18 Aug demo.

## 5. Device permissions & capabilities

**Internet only.**

| Platform | Allowed |
|----------|---------|
| Android | `INTERNET`, `ACCESS_NETWORK_STATE` (manifest, not runtime) |
| iOS | No usage-description strings for camera, location, contacts, mic, photos, Face ID, local network, or photo library |

Cast and downloads chrome stays inert (doc 02). Biometrics wait for real JWT (**Phase 3**; doc 16 deferred Face ID). Denied-permission degradation is N/A until a later phase adds a capability.

**Forecloses:** Face ID, AirPlay/cast, downloads, camera, location in Phase 1.

## 6. Device security

Demo-grade. No real PII, no TLS host (I/O is in-process mocks).

| Control | Phase 1 | Later |
|---------|---------|-------|
| Token at rest | Keychain / EncryptedSharedPreferences via encrypted-storage | same engine; JWT is a payload change |
| Certificate pinning | **None** | API-module interceptor when a real host exists (Phase 3) |
| Jailbreak / root detection | **None** (false-positives on simulators and debug builds) | store-bound release |
| Transport | N/A (mocks) | HTTPS only, ATS on, no cleartext exception |
| Logging | Tokens never logged | same |
| Extra encrypted DB | None | only if a later phase stores more than the auth slice |

See RISK-0007. Security session 1.6 is **Done** (abbreviated) in `architecture/06-security-threat-model.md`; it does not redo token storage.

**Forecloses:** MITM defenses and compromised-device lockout on 18 Aug.

## 7. Distribution & release

**Local installs only.** Xcode / Android Studio → simulator or USB device. No App Store, Play Store, TestFlight, Play internal track, or OTA (no Expo; CodePush is dead).

**The sign-off artifact is a release build on a device** — declared, not contingent (doc 08 §2). Debug builds are the development loop; the demo runs an embedded JS bundle with no Metro attached. Procedure: `runbooks/release-deploy.md`.

Stamp `CFBundleShortVersionString` / `versionName` so a later min-version / forced-upgrade check can live in the shell without a native rewrite. No forced-upgrade gate and no phased rollout until there is a fleet.

Phase 2 still owns Bitrise + installable QA builds. **RISK-0005:** do not submit **QC+** to any store without legal review.

**Forecloses:** OTA JS patches and remotely killing old binaries on 18 Aug.

## 8. Device performance & storefront pagination

**No numeric SLOs** (binary size, cold start, battery, FPS, memory). Doc 01 already dropped performance from v1 success criteria. If a device stutters, that is craft to polish, not a performance program to open — and the sign-off already runs a **release** build by decision (§7, doc 08 §2), so it is not a remedy held in reserve.

### Guardrails

- **Hermes** on.
- **Virtualized lists** for carousels — recycled rows, not a wall of `Image`s.
- Placeholder art only at the target aspect ratios (`architecture/assets/placeholder-art/`).
- No background fetch, no video SDK, no push/Firebase native blobs.

What would blow it: unbounded image decode, persisting the RTK Query cache, shipping the debug bundle to the sign-off device.

### Storefront pagination

The storefront **paginates**. Screens do not fetch; a **storefront-owned hook** (or a small pair of hooks) is the organized API for page state.

| Surface | What pages | Owner |
|---------|------------|--------|
| Home feed (vertical) | Next page of **rows / carousel configs** | Storefront hook, e.g. `useHomeFeed` |
| Carousel (horizontal) | Next page of **tiles** in that row | Storefront hook, e.g. `useCarouselPage` |

Each hook exposes at least `{ items, loadMore, hasMore, isLoading, error }`. `loadMore` is an RTK Query fetch (`getHomeFeed` / `getContainerResources`). Mocks return paginated pages with artificial latency and can fail (DF2). Pagination cursors live in the RTK Query cache — not in Keychain.

Virtualized lists call `loadMore` on end-reached. Do not load every tile in every row on first paint.

**Wire format** is locked as opaque **`nextCursor`** (doc 04), and **doc 11 §6.1–§6.2 names the envelope**: `{ data, nextCursor }`, with a cursor at *each* paging level — the page envelope's cursor pages containers, each Container's own cursor pages its `resources`. HomeFeed first page is **`Container[]`** (hero + 15); ContinueWatching is a **second** `Container[]` with `variant: "progress"` (ADR-0006 / ADR-0007). **Page sizes (OQ-23 closed):** `limit` defaults to 16 on HomeFeed, 10 on Continue Watching and on `resources`.

`useCarouselPage` fetches **`GET /containers/{containerId}/resources`** — an operation this table
implied but no doc named until 1.11 (doc 11 §5).

**Forecloses:** treating FPS/size as an 18 Aug gate; dumping the full catalog into one mock payload.

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Platform | Bare RN, iOS + Android, no Expo | Already locked in 01/03; one codebase for the two-platform demo | Dual native, Flutter, Expo/OTA, web/desktop |
| 2 | Offline | Online-only; no content cache | Downloads are Phase 4+; mocks are the network | Offline browse / last-known-home |
| 3 | Connectivity UX | Shell overlay + NetInfo (**interface up = online**, ADR-0014); auto-restore by auth state; `REINTENTAR` | Matches the Disney+ reference; one gate, not per-feature screens; a captive portal cannot block a mocks-only demo | Per-screen offline UIs; treating “no network” as a fetch error; detecting captive portals in Phase 1 |
| 4 | Secure storage | `react-native-encrypted-storage` + redux-persist, auth slice only | npm Keychain/Keystore wrapper; persist API already matches | AsyncStorage; custom native module; `react-native-keychain` adapter |
| 5 | Push | None in Phase 1 | No backend, no vendor | Demo pushes |
| 6 | Permissions | `INTERNET` + `ACCESS_NETWORK_STATE` only | Nothing else is exercised | Biometrics, cast, downloads, camera, location |
| 7 | Device security | Demo-grade: no pinning, no root detection | Mocks never leave the process; simulators false-positive | MITM / compromised-device controls until Phase 3 / store |
| 8 | Distribution | Local Xcode / Android Studio only | Internal demo; no fleet | TestFlight / Play / OTA / forced upgrade in Phase 1 |
| 9 | Performance | No numeric SLOs; virtualized lists; Hermes | POC on a handful of devices | FPS/size as a sign-off criterion |
| 10 | Storefront pagination | Feature hooks + paginated mocks; opaque `nextCursor` (doc 04); two `Container[]` endpoints (ADR-0006 / ADR-0007) | Keeps load organized; silent CW reload | Loading the full feed on first paint |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-19~~ | ~~Pagination wire format: cursor vs offset~~ **Resolved (1.4):** opaque `nextCursor`. JSON names → 1.11 (OQ-22) | — | closed |
| ~~OQ-20~~ | ~~Home first-page size and per-carousel page size~~ **Resolved in part (1.4):** HomeFeed first page = hero + 15; CW separate. Tiles per row → OQ-23 | — | closed (partial) |
| ~~OQ-21~~ | ~~NetInfo “usable network”: treat cellular+wifi as enough, or also require internet reachability (vs captive portal)?~~ **Resolved (1.8):** interface up is enough; no reachability probe (ADR-0014) | — | closed |
| OQ-29 | Does “connected but not reachable” need a reachability probe once a real host exists? | Mobile | Phase 3 backend integration (doc 08 §6.1) |

**OQ-16** is closed (encrypted-storage). **OQ-17** is closed then reversed (1.14 / ADR-0020): **`axios-mock-adapter` on the real instance** so interceptors run. Both transports still normalize to one
`ApiError` (doc 11 §6.5, §8.3). **OQ-23** is closed (doc 11 §6.3).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-16 | STEP-1.3a | Initial draft from the native-app session |
| v0.2.0 | 2026-08-16 | STEP-1.4 | User slice memory-only; `nextCursor`; two feed endpoints. Closed OQ-19 / OQ-20 (partial). |
| v0.2.1 | 2026-08-17 | STEP-1.5 | Feeds are `Container[]` / `resources: Card[]`; CW `progress` variant. |
| v0.2.2 | 2026-08-17 | STEP-1.6 | Device-security deferrals indexed from doc 06 (ADR-0008); token storage unchanged. |
| v0.2.3 | 2026-08-17 | STEP-1.6a | Biometrics stay Phase 3 (doc 16); no Face ID in Phase 1. |
| v0.2.4 | 2026-08-17 | STEP-1.8 | §2 connectivity gate keys on interface state, not reachability (ADR-0014). Closed OQ-21; opened OQ-29. §7 distribution: the release build is now the declared sign-off artifact (doc 08 §2). |
| v0.2.5 | 2026-08-17 | STEP-1.11 | §8 envelope and page sizes locked from doc 11; `useCarouselPage`'s operation named (`GET /containers/{id}/resources`). Closed OQ-17, OQ-23. |
| v0.2.7 | 2026-08-18 | STEP-6.1 | Native rebrand: **`QCPlusApp`** / **`com.qcplus.app`**, display **`QC+`**, stakeholder icon pack wired (iOS AppIcon, Android mipmap, splash, `qc-plus-mark.svg`). |
