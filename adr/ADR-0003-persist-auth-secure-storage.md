# ADR-0003: Persist the auth slice to platform secure storage from day one

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/03-architecture-overview.md`
- `architecture/02-phasing-roadmap.md` (DF3)
- `architecture/15-native-app-architecture.md`

## Context

A signed-in session must survive process death so the demo does not bounce to Welcome on
every cold start. DF3 already required an opaque token in the auth slice and that secure
storage must plug in without touching screens. Using AsyncStorage for a fake token would
force a storage rewrite the moment the token is real.

## Decision

1. **redux-persist** rehydrates **the auth slice only** (opaque token / `isAuthenticated`).
   Storefront, errors, and UI flags are not persisted.
2. **The persist storage engine is Keychain (iOS) / Keystore (Android) from day one**, via
   `react-native-encrypted-storage` or a thin Keychain adapter — **not AsyncStorage**.
3. Screens never call the secure-store API; they read the auth slice after rehydrate.
4. The app shell waits for rehydrate before choosing the auth vs app navigator.

**Rationale.** A real JWT later is a payload change inside the same slice and engine, not a
persistence rewrite. Whitelisting auth keeps Keychain-sized secrets out of a full-store blob.

**Alternatives.** No persistence — rejected; cold start would always show Welcome.
AsyncStorage now, Keychain later — rejected; that is the rewrite DF3 exists to avoid.
Persisting the whole store — rejected; content and error UI are not secrets and would bloat
secure storage.

**Reversibility.** Low if the whitelist and engine stay. High cost if Phase 1 ships
AsyncStorage and must migrate tokens later.

## Consequences

- Foundation STEP must wire persist + a secure storage backend before auth UI.
- Native-app / Security sessions still own pinning, jailbreak, and refresh; they do not
  redo token storage.
- Exact library (`react-native-encrypted-storage` vs Keychain adapter) is OQ-16.

## Amendment (2026-08-16 — STEP-1.3a)

OQ-16 is **closed**. The persist storage engine is **`react-native-encrypted-storage`**
(Keychain on iOS, EncryptedSharedPreferences on Android). A `react-native-keychain`
adapter was the rejected alternative. Decision items 1–4 above are unchanged.

## Amendment (2026-08-16 — STEP-1.4)

The **user slice is not persisted**. Profile (`userName`) is memory-only and filled by an
authenticated `/me` on every cold start and after login (see `architecture/04-data-model.md`).
Decision item 1 still means **auth slice only** — do not add the user slice to the
redux-persist whitelist.
