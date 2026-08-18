# Doc 09 — Environments

**Version:** v0.1.3
**Status:** Draft
**Coverage:** full for Phase 1. A `staging` tier and CI-as-environment are consciously deferred
with named triggers (§2.1, §8) rather than left unenumerated.
**Last updated:** 2026-08-18 (STEP-2.2)
**Audience:** Mobile developers, QA, eng leadership

> What "environment" means in a project that deploys nothing, how config and secrets reach the
> app, and what has to be true about the device before a build is worth showing anyone.

## Table of Contents

1. [An environment here is a build configuration, not a destination](#1-an-environment-here-is-a-build-configuration-not-a-destination)
2. [The two configurations](#2-the-two-configurations)
3. [No sandbox](#3-no-sandbox)
4. [Config & secrets](#4-config--secrets)
5. [Data & device state](#5-data--device-state)
6. [Parity — the release-only failure class](#6-parity--the-release-only-failure-class)
7. [Promotion flow](#7-promotion-flow)
8. [Access control](#8-access-control)

---

## 1. An environment here is a build configuration, not a destination

The usual local → staging → production ladder assumes somewhere to deploy. Doc 08 §1 is
unambiguous that there is nowhere: no server, no database, no hosting tier, no store release.
All Phase-1 I/O is in-process mocks, and the artifact is a release build installed over USB
onto a device someone is holding.

So the ladder does not survive contact with this project, and reproducing it would produce
tiers that differ from one another in nothing. What remains, and what this doc governs, is the
narrower and real thing:

> An **environment** in quasar-disney-mobile is a **named build configuration plus the config
> values the app is built with**. There are two of them, and they map onto the schemes and build
> types React Native already provides.

This is not a smaller version of the standard model. It is a different model, and the parts of
the standard model that *do* still bite — config drift between machines, code that only breaks
in release, a device in the wrong state — are handled in §4–§6 rather than by inventing tiers.

## 2. The two configurations

| Environment | Purpose | Who has access | Data |
|-------------|---------|----------------|------|
| **`development`** | The daily loop. Metro-served bundle, simulator or tethered device, `__DEV__` on — logging, Redux DevTools, fast reload | Both developers, on their own machines | The same committed fixtures as `release` (§5) |
| **`release`** | **The artifact stakeholders see.** JS bundle embedded, `__DEV__` stripped, no Metro, installed over USB (doc 08 §2) | Whoever holds the demo device; built by **Raul Angel** (OQ-28 closed) | Identical fixtures; **no persisted session** at demo start (§5.1) |

They correspond to RN's existing `Debug`/`Release` iOS schemes and `debug`/`release` Android
build types. **No product flavors, no additional schemes, no app-id suffixes** — the two configs
that already exist are named and given meaning rather than supplemented.

**CI is not a third environment.** When Bitrise lands in Phase 2 (doc 08 §2, externally blocked
on OQ-05), it is a *runner* of the `release` configuration plus the test suite — it gets a row
in the promotion flow (§7), not a row in this table. It has no distinct config, no distinct
data, and no distinct audience. See **OQ-32** for the one thing that could change that.

### 2.1 The deferred `staging` tier

A third tier was considered and consciously rejected for Phase 1, on the grounds that it would
be **byte-identical to `release`**: the same mocks, the same fake credentials, and an
`API_BASE_URL` pointing at nothing, because there is nothing to point at. A tier that differs
from another tier in nothing teaches nothing and rots quietly until someone trusts it.

**Revisit trigger — Phase 3 backend integration.** The moment `API_BASE_URL` addresses a real
host, a `staging` configuration becomes meaningful, because "which host" becomes a real
difference. That is also the moment it likely needs *native* build config — a distinct
application id so a staging and a production build can sit side by side on one device — which
reopens the mechanism choice in **ADR-0015**. Recorded as **OQ-31**.

## 3. No sandbox

**Decision: no sandbox or demo environment.** For most products the question is real — somewhere
to try the thing without touching live data, or for external integrators to test against. Here
it inverts:

- **The demo *is* the product.** `release` exists precisely to be shown to stakeholders; a
  separate "demo" tier would be a synonym for it.
- **There is no real data to protect a sandbox from.** Credentials are fake by construction
  (doc 06 §5), the catalog is committed fixtures, and no request leaves the process.
- **There are no external integrators.** The Phase-3 backend team consumes the *contract*
  (doc 11, OQ-10), not a running instance of this app.

**Revisit trigger:** external parties needing a build against a non-production backend — the
same Phase-3 trigger as the deferred `staging` tier (§2.1).

## 4. Config & secrets

Docs 06 §5 and 08 §5 both assert that the release build reads the demo credentials from `.env`.
Neither names a mechanism, and React Native has no native `.env` reader — `process.env` is not
populated at runtime on device. This session closes that gap.

### 4.1 Mechanism

**`react-native-dotenv`**, a Babel plugin, resolving `import { DEMO_EMAIL } from '@env'` at
transform time. Values are inlined into the JS bundle, so they are present in the release build
identically to the development build. See **ADR-0015** for the alternatives weighed and what
this forecloses.

- It is a **build-time dev dependency**, not a runtime one, so doc 03 §7's hard-dependency list
  (completed in 1.7 by ADR-0013) is **unchanged**.
- TypeScript needs a `declare module '@env'` declaration file, checked in. This is the one piece
  of friction the choice carries.

### 4.2 One `.env`, not one per configuration

**A single gitignored `.env` serves both configurations.** The values genuinely do not differ:
same in-process mocks, same fake credentials, and `API_BASE_URL` is inert in Phase 1 with no
host to address. Splitting it into `.env.development` and `.env.release` would create two files
that must be held identical by hand — a drift source wearing the costume of a safety mechanism.
It also keeps doc 08 §5's pre-flight check a single item instead of two.

| Key | Role | Phase 1 value |
|-----|------|---------------|
| `API_BASE_URL` | Base URL for the axios instance | Inert — no host exists. Carried so Phase 3 is a value change, not a code change |
| `DEMO_EMAIL` | The mock adapter's accepted account | Fake |
| `DEMO_PASSWORD` | The mock adapter's accepted password | Fake |

The mock adapter reads the demo pair from env — **not** a committed fixture and **not** a
screen-level `if` (doc 06 §5, unchanged).

`.env.example` is committed with placeholder values and one line of documentation per key. It
lives in `Code/quasar-disney-mobile-app/.env.example` (created STEP-2.2) from the convention
in `templates/env-example.txt`. Its three keys are the table above.

### 4.3 Differences between configurations are expressed in code, not config

Nothing in `.env` differs between `development` and `release`. What differs — verbose logging,
Redux DevTools attachment, any development-only affordance — keys off **`__DEV__`**, which the
release build strips at compile time. This keeps the environments table honest: two
configurations, one config file, and differences the compiler can prove are absent from the
shipped binary.

### 4.4 Standing rule: nothing in a mobile binary is a secret

Under `react-native-dotenv` the values are inlined into the JS bundle. Under
`react-native-config` they would sit in the native config. Under a hand-rolled module they would
be in the source. **All three ship inside the binary and are recoverable by anyone holding the
app.**

In Phase 1 this is harmless: the credentials are fake, and there is no backend for them to
unlock. But this project exists as a migration template (doc 01), and "we put it in `.env`" must
not be learned as "it is therefore secret." The rule to carry forward:

> `.env` keeps secrets **out of version control**. It does not keep them out of the **artifact**.
> A value that must stay secret in a real product belongs server-side, behind the API — never
> compiled into a mobile client, by any mechanism.

Phase 3's real credentials are therefore an API-side concern, consistent with ADR-0009's
buy-behind-our-API posture for the identity provider.

## 5. Data & device state

**One fixture set, identical in both configurations.** The catalog source is already locked by
doc 04 §7 — an in-process mock adapter with TS/JSON fixtures inside the API module, HomeFeed's
first page being one `hero` container plus 15 others, with Continue Watching as a separate
authenticated fetch. This session adds only that `development` does not get a reduced set.

The reasoning is that `release` *is* the demo: developing against a trimmed fixture set would
mean the first time anyone sees real content volume is on stage, and it would hide whether
ADR-0005's pagination is genuinely exercised. Fixtures are compiled TypeScript either way, so a
smaller set saves nothing worth having.

### 5.1 The demo device must start logged out

ADR-0003 persists the auth slice to platform secure storage, and doc 04 sets the mock JWT's
`exp` at **7 days** with no refresh flow. Followed through to sign-off, that produces a silent
failure mode:

> Anyone who logs in during rehearsal — the Tuesday sync point, a smoke test, a "does this build
> work" check — leaves a valid persisted session on the device. The token is still days from
> expiry on 2026-08-18. The app therefore **launches straight into the storefront**, skipping
> welcome → email → password → inline error entirely.

That flow is roughly half of what Phase 1a exists to demonstrate (doc 02 §4, criteria F1–F2).
Every component behaves exactly as designed; the demo just quietly omits the part the audience
came for. Doc 08 §6's SPOF table covers the device, the laptop, the `.env` file, and the venue
network — it did not cover **device state**.

**Decision: install to a clean state, and verify it.** The demo build goes onto a device with no
prior session — uninstall first, or use a device that has never run it — confirmed by observing
that the app opens on the **welcome screen**. This is a checkbox in
`runbooks/release-deploy.md` and a row in doc 08 §6, not a code change.

A `__DEV__`-gated "clear session" control was considered and rejected: it is compiled out of the
release build precisely when it would be wanted, and building one that survives into release
would contradict §6.2.

## 6. Parity — the release-only failure class

There is no production to be at parity *with*. The parity question this project actually has is
between its own two configurations, and it is not cosmetic. `release` differs from `development`
in ways that routinely break apps that worked all week:

| Difference | Failure it produces |
|------------|---------------------|
| JS bundle embedded rather than Metro-served | Assets that resolved via the dev server are missing |
| `__DEV__` blocks stripped | Any logic that drifted inside one silently disappears |
| `console.*` removed | Code depending on a side effect of logging changes behavior |
| Hermes compiles ahead of time rather than at runtime | Different failure timing and stack traces |

### 6.1 Smoke `release` at every sync point, not once

Doc 08 §2 puts release-build *setup* at the Tuesday-AM sync point, which gets it built one time.
That is not sufficient: the gap between "the release build works" and "the release build works
**with the final code in it**" is exactly where this failure class lives. **The release
configuration is built and smoked at every sync point in doc 02 §8**, not only at setup.
Discovering a release-only break on Tuesday is recoverable; discovering it on the night of the
17th is not.

**Android minification stays off** for Phase 1 (`enableProguardInReleaseBuilds = false`, the RN
template default). It is one fewer release-only variable before a fixed date, and there is no
size or IP pressure on an internal POC that would justify carrying it.

### 6.2 Two things are called "failure" and only one ships

| | What it is | Where it exists |
|---|-----------|-----------------|
| **Deterministic failure** | Wrong credentials are rejected by the mock adapter | **In `release`, always.** Doc 02 criterion **F2** requires the inline error state on stage |
| **Injectable failure** | Forcing HomeFeed or Continue Watching to fail, to exercise the storefront error state | **Test-time only.** Consumed by the tests specified in `architecture/12-test-strategy.md` §3.3 |

Doc 03 §7 gives the mock adapter "latency and injectable failure" and doc 05 §47 sets the
artificial latency at **400–600 ms** so loading and error paths are real (doc 02 DF2). Both
stand. What this session adds is the boundary: **injection is a test seam, not a runtime toggle
— there is no debug menu, shake gesture, or hidden control in the release binary.** Without
that line written down, someone reasonably concludes the demo build needs a way to trigger the
error state, and ships a debug affordance into the artifact stakeholders hold.

**Extended in 1.12.** The adapter has **three** such seams, all constructor parameters and none a
runtime toggle: failure injection, **latency** (defaulting to the 400–600 ms above; zero in tests so
the suite stays fast), and the **clock** (defaulting to `Date.now`; frozen in tests so `exp` cases are
deterministic). Doc 12 §4.4 owns the detail, and doc 12 §3.3 adds a test asserting the *default*
latency is nonzero and inside this band — because the likeliest way DF2 quietly dies is someone
zeroing the default after seeing it zeroed in tests.

## 7. Promotion flow

With no deployment tiers, promotion is how code and config reach the sign-off binary. Doc 08 §3
already fixes the mechanics — tag → stamp → clean → build → install → smoke → keep the previous
binary. Two things it left open:

### 7.1 Release builds are cut from trunk only

```
step-NNNN-* branch  →  review / PR  →  merge to trunk  →  git tag
                    →  release build from the tagged trunk commit  →  install  →  smoke
```

**Never from a `step-NNNN-*` branch.** Doc 02 §9 splits two developers across a Sat/Sun/Mon
seam, so the sign-off binary has to contain both halves — and a branch build that happens to
work is the single most tempting shortcut available on the 17th, as well as the one most likely
to produce a binary nobody can later reconstruct. The git tag doc 08 §3 already requires is what
makes the artifact traceable, and it can only be trusted if the build came from the tagged
commit.

Phase 2 inserts Bitrise into this line as a *runner* — merge to trunk triggers tests and a
`release` build — without adding an environment.

### 7.2 Config propagates by hand, so `.env.example` is the contract

Code moves through git. **`.env` does not** — it is gitignored, so it travels machine to machine
by hand. Doc 08 §5's pre-flight check verifies that `.env` *exists* on the build laptop. It does
not verify that it is *complete*, and with two developers that gap is live:

> Dev B adds a key while building the mock adapter and updates their own `.env`. Either
> `.env.example` is not updated, or it is and Dev A's local `.env` goes stale. The next build
> succeeds and misbehaves at runtime — the Babel transform inlines `undefined` — surfacing as a
> login that fails for the wrong reason. Precisely doc 08 §5's scenario, one step upstream.

Two rules close it, both free:

1. **`.env.example` is updated in the same PR as any code that reads a new key.** Adding a key
   without adding it there is an incomplete change, reviewable as such.
2. **The pre-flight check verifies `.env` has every key in `.env.example`** — not merely that
   the file exists. A manual comparison or a three-line script; strictly better than the check
   currently written down, at no additional cost.

### 7.3 Who deploys

**OQ-28** (doc 08) is **closed:** **Raul Angel** owns building and installing the sign-off
binary (STEP-6). The two demo devices are named at the STEP-6 pre-flight. §5.1 and §6.1 still
give that pre-flight a dated sequence that owner has to run before the 18th.

## 8. Access control

There is no production tier, no secrets manager, and no deployed surface, so "who can access
production" has no referent. What is actually access-controlled, and whether it warrants a rule:

| Surface | Phase 1 reality | Rule |
|---------|-----------------|------|
| Repos | Both developers, via the git host's own permissions | None — not this doc's concern |
| Demo device(s) | Physical possession is full access | Covered by the clean-state pre-flight (§5.1) |
| `.env` values | Fake credentials; blast radius is zero | The **channel** gets a rule, below |
| Production | Does not exist | N/A |

The only item with substance is how `.env` moves between the two laptops, since §7.2 establishes
it propagates by hand. A two-tier rule, for the same migration-template reason as §4.4:

- **Phase 1:** any channel is fine. The values are non-secret by construction, and pretending
  otherwise is theatre.
- **Standing rule:** real secret values move through a password manager or secrets manager —
  **never chat, never git.** "We DM'd the `.env`" is exactly the habit that survives into a
  project where the values are not fake.

If anything real is ever suspected leaked, the escalation path is
`runbooks/secrets-rotation.md` Part 2 (doc 06 §5).

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | What an environment is | A **named build configuration + its config values** — not a deployment destination | Doc 08 §1 deploys nothing; a local/staging/prod ladder would produce tiers identical to each other | Reusing standard tier vocabulary; any per-tier infrastructure thinking |
| 2 | Which environments | **Two: `development` and `release`**, mapping onto RN's existing Debug/Release schemes and build types | The configurations already exist — this names them rather than adding flavors | Product flavors, extra schemes, app-id suffixes, side-by-side installs |
| 3 | CI as an environment | **No — CI is a runner** of the `release` config plus tests; it gets a promotion-flow row, not an environments row | No distinct config, data, or audience of its own | Revisit if Bitrise needs its own secret source (**OQ-32**) |
| 4 | `staging` tier | **Deferred**, with the shape and trigger recorded (§2.1) | It would be byte-identical to `release` today: same mocks, same fake creds, `API_BASE_URL` pointing at nothing | Rediscovering the need at Phase 3 instead of finding it written down (**OQ-31**) |
| 5 | Sandbox | **None.** The demo *is* the product; there is no real data and no external integrator | A sandbox tier would be a synonym for `release` | Revisit at external parties on a non-production backend (Phase 3) |
| 6 | Config mechanism | **`react-native-dotenv`** — Babel transform, `@env` module, JS-only (**ADR-0015**) | Its rival's advantage is native build config, which decision 2 just made unnecessary | Native/per-flavor config; needs a `declare module '@env'` file; reopens at Phase 3 |
| 7 | Dependency list | **Doc 03 §7 unchanged** — this is a build-time dev dependency, not a runtime one | Not comparable to ADR-0013's runtime additions | — |
| 8 | `.env` layout | **A single gitignored `.env`** for both configurations; three keys (§4.2) | The values genuinely don't differ; two files would be a drift source, not a safeguard | Per-configuration value differences (revisit with `staging`) |
| 9 | Config differences | Expressed via **`__DEV__`**, not config keys | The compiler proves them absent from release | Runtime-switchable behavior in the shipped binary |
| 10 | Secrets in the artifact | **Standing rule: nothing in a mobile binary is secret**, under any mechanism | `.env` protects version control, not the artifact — and this project is a migration template | Training "put it in `.env`" as a security control |
| 11 | Fixture data | **One set, identical in both configurations** | The demo *is* `release`; a trimmed dev set hides content volume and pagination behavior | A faster/lighter development dataset |
| 12 | Device state | **Install to a clean state; verify the app opens on the welcome screen** | ADR-0003 persistence + a 7-day mock `exp` means a rehearsal login silently deletes half the demo | A `__DEV__` "clear session" control — stripped from release exactly when wanted |
| 13 | Parity cadence | **Build and smoke `release` at every sync point**, not only at setup | "Release works" ≠ "release works with the final code in it" | Treating the Tuesday setup as sufficient |
| 14 | Minification | **Off** in Phase 1 (`enableProguardInReleaseBuilds = false`) | One fewer release-only variable before a fixed date; no size or IP pressure | Revisit only at a store release |
| 15 | Failure injection | **Deterministic failure ships; injectable failure is test-time only** | F2 needs wrong-credentials rejection on stage; error-state injection is a test seam | A debug menu, shake gesture, or hidden toggle in the demo binary |
| 16 | Promotion | **Release builds cut from trunk only**, on the tagged commit — never from a `step-NNNN-*` branch | Two developers across a seam; the tag is only trustworthy if the build came from it | Convenience branch builds on the 17th |
| 17 | Config promotion | **`.env.example` updated in the same PR** as any new key; **pre-flight verifies completeness**, not existence | The Babel transform inlines `undefined` silently — it fails at runtime, looking like an auth bug | Relying on the existing "file exists" check |
| 18 | Access control | Repos/devices need no new rule. **Standing rule:** real secrets move via a password/secrets manager, never chat or git | Phase 1 values are non-secret by construction; the *habit* is what carries forward | Treating the Phase-1 posture as the template's security model |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-31 | At Phase 3, does the `staging` configuration need **native** build config (distinct application id, side-by-side install) — reopening ADR-0015's mechanism choice in favor of `react-native-config`? | Mobile | Phase 3 backend integration |
| OQ-32 | Does Bitrise need its own secret source for `.env` values (a CI secret-env entry), and would that make CI an environment row rather than a runner? | Mobile | Phase 2 Bitrise STEP (OQ-05) |

Carried forward: **OQ-05** (Bitrise setup) and **OQ-27** (closed — audit gates deferred to Bitrise implementation)
are unchanged. **OQ-28** is closed (planning session: Raul Angel owns the sign-off binary).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.9 | Initial draft from the environments session. Two build configurations named; sandbox and `staging` declined with triggers; config mechanism closed via ADR-0015; clean-state pre-flight and `.env` completeness check added (doc 08 §5–6 and `runbooks/release-deploy.md` updated); release-build parity cadence and the deterministic-vs-injectable failure boundary set. Opened OQ-31, OQ-32. |
| v0.1.1 | 2026-08-17 | STEP-1.12 | §6.2 extended: the adapter carries **three** test seams — failure injection, latency, and the clock — all constructor parameters, none a runtime toggle (doc 12 §4.4). Also recorded there: **tests never read the real `.env`**, resolving `@env` to a committed stub instead, which is what makes §7's CI tier possible on a runner that can never have a gitignored file (ADR-0018). No change to the two configurations, the `.env` key set, or the promotion flow. |
| v0.1.2 | 2026-08-17 | planning session | Closed OQ-28 in §7.3: Raul Angel owns the sign-off binary. |
| v0.1.3 | 2026-08-18 | STEP-2.2 | `.env.example` now exists in `Code/quasar-disney-mobile-app/`. No change to the two configurations or the key set. |
