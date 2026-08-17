# ADR-0015: Config reaches the app through a Babel transform, not native build config

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- architecture/09-environments.md §4 (config & secrets), §2.1 (deferred `staging` tier)
- architecture/06-security-threat-model.md §5 (secrets & data protection)
- architecture/08-infrastructure-deployment.md §5 (networking, TLS, secrets)
- architecture/03-architecture-overview.md §7 (build vs. buy — hard dependencies)
- ADR-0013 (dependency-list completeness), ADR-0009 (IdP buy-behind-API)

## Context

Docs 06 §5 and 08 §5 both state that the release build reads `DEMO_EMAIL` / `DEMO_PASSWORD`
from a gitignored `.env`, and doc 08 §6 lists a missing `.env` as a single point of failure for
the 2026-08-18 sign-off. Neither doc names a mechanism.

React Native has no native `.env` reader. `process.env` is not populated at runtime on device,
so as written the assertion had nothing behind it — a gap of the same shape as the one ADR-0013
found between "we chose SVG assets" and "we listed our libraries", and one that would have
surfaced during the scaffold window that RISK-0003 already flags as the critical-path schedule
risk.

Session 1.9 had also just decided there are exactly **two build configurations**, `development`
and `release`, with no product flavors and no app-id suffixes (doc 09 §2). That decision is what
makes the mechanism choice tractable, because it removes the main reason to reach for the
heavier option.

## Decision

**Adopt `react-native-dotenv`** — a Babel plugin that resolves `import { KEY } from '@env'` at
transform time, inlining values into the JS bundle. It is a **build-time dev dependency**, so
doc 03 §7's hard-dependency list (completed by ADR-0013) is unchanged. A checked-in
`declare module '@env'` declaration file satisfies TypeScript.

**Alternatives.**

- ***`react-native-config`*** — the more conventional choice, and the one that reaches **native**
  build config (Gradle, `Info.plist`), enabling per-flavor application ids and display names. Its
  entire advantage is capability that doc 09 §2 just decided against building. It costs a native
  dependency, a `pod install`, a Gradle plugin, and an amendment to doc 03 §7 — paid for
  capability nothing in Phase 1 uses.
- ***No library*** — a gitignored `src/config/local.ts`. Contradicts docs 06 §5 and 08 §5, which
  name `.env` explicitly, and trains a convention the team would have to unlearn.

**Reversibility.** High. Both libraries are consumed through an import in the API module; the
call sites are few and mechanical. Switching is a dependency swap plus a Babel/Gradle config
change, not a code-architecture change.

## Consequences

**Easier.** The `.env` posture docs 06 and 08 already assert becomes true rather than aspirational,
at zero native cost, in a scaffold window that cannot absorb surprises. Nothing in doc 03 §7 has
to move.

**Harder.** No native build config, so per-flavor application ids and side-by-side installs are
unavailable without revisiting this. That is exactly the capability a Phase-3 `staging` tier is
likely to want — recorded as **OQ-31** with this ADR named as the thing to reopen, rather than
left for someone to rediscover.

**Constraint this makes explicit.** Values are inlined into the JS bundle and therefore ship
inside the artifact, recoverable by anyone holding the app. This is **not a property of the
choice** — `react-native-config` and a hand-rolled module have it too. It is a property of
mobile clients, and it is stated here because the project is an intended migration template
(doc 01): `.env` keeps secrets out of **version control**, not out of the **artifact**. Phase 1
is unaffected — the credentials are fake and there is no backend to unlock — and the forward
rule (doc 09 §4.4) is that anything that must stay secret in a real product lives server-side
behind the API, consistent with ADR-0009.
