# Doc 12 — Test Strategy

**Version:** v0.1.3
**Status:** Draft
**Coverage:** full for Phase 1a. UI/component tests (1b), automated device e2e (Phase 2), and
numeric coverage gates (1b) are consciously deferred with named triggers (§2, §5, §8) rather than
left unenumerated.
**Last updated:** 2026-08-18 (STEP-3.5)
**Audience:** Mobile developers, QA, backend team (Phase 3)

> What gets tested in a codebase whose network layer is a fixture, what has to be green before code
> merges, and which architecture criteria stop being review habits and become machine-enforced.

## Table of Contents

1. [What testing is for here](#1-what-testing-is-for-here)
2. [Test tiers](#2-test-tiers)
3. [Coverage priorities](#3-coverage-priorities)
4. [Test data & isolation](#4-test-data--isolation)
5. [Mocking strategy](#5-mocking-strategy)
6. [System / end-to-end testing](#6-system--end-to-end-testing)
7. [CI gates](#7-ci-gates)
8. [Coverage tooling and reporting](#8-coverage-tooling-and-reporting)
9. [Performance & load testing](#9-performance--load-testing)
10. [Coding standards](#10-coding-standards)
11. [Scaffold obligations](#11-scaffold-obligations)

---

## 1. What testing is for here

Doc 02 criterion **A6** already requires green tests for *every reducer, RTK Query endpoint path, hook, and the
API/mock layer*.
API/mock layer*, and **A1–A5** name four more architectural properties the demo is supposed to
prove. This session's job was not to invent a test policy — it was to decide where the tier lines
fall, what the must-cover list actually contains, and which of A1–A6 can be made mechanical instead
of trusted to review under deadline pressure.

Two things about this codebase shape every decision below.

**The mock adapter is a production artifact, not a test double.** Doc 11 §6.5 makes it one of two
transport implementations of a single contract, and doc 09 §6.2 makes its deterministic failure
path a *demo* requirement (criterion F2). So the thing you would normally mock is here one of the
primary systems under test. That inverts the usual mocking question (§5) and it is why the
must-cover list (§3) is heaviest around `src/api/`.

**Roughly half the launch criteria are not screen behavior.** A2 (no hardcoded design values), A3
(API swap), A5 (no cross-feature imports) and doc 03 §8.2's import rules are structural properties.
A test suite cannot assert most of them, but a linter can assert several — and a rule that runs on
every push is worth more than a convention two developers agree to on a Friday and then work
against separately all weekend (doc 02 §9).

## 2. Test tiers

| # | Tier | Scope | Tools | Where it runs | Phase |
|---|------|-------|-------|---------------|-------|
| **T0** | **Type check** | `tsc --noEmit` across `src/`. This is the contract test (doc 11 §11.2) — it is only meaningful because fixtures are typed `Container[]` / `Card[]` and never `any` | TypeScript `strict` | Local + CI | **1a** |
| **T1** | **Unit** | Pure logic in isolation: reducers, selectors, the mock adapter's rules (credential compare, `exp` validation, cursor exhaustion), `ApiError` normalization, i18n formatters, home composition | Jest (React Native preset). **No renderer** | Local + CI | **1a** — the bulk |
| **T2** | **Integration** | Real store + `baseApi.middleware` + axios instance against `axios-mock-adapter`; hooks via `renderHook` once a feature owns one. Doc 11 §11.3's behavioral tests live here | Jest. **RNTL arrives with the first feature hook** (STEP-4/5) — see below | Local + CI | **1a** — thin, load-bearing |
| **T3** | **Component render** | Screens and atoms render from theme tokens; a11y props present (RISK-0011) | React Native Testing Library | Local + CI | **1b** |
| **T4** | **Device e2e** | — | **Declined** — see below | — | — |

**T2 landed without RNTL, and that is the right order** (STEP-3.5). The API module has no hooks of
its own: `login` / `getMe` belong to Auth and the feed operations to Storefront (doc 03 §8.2), so
the T2 suite injects **test-local endpoints** onto `baseApi` and drives them with
`store.dispatch(endpoint.initiate(...))`. That exercises the whole path — store → middleware →
`baseQueryWithAuth` → interceptors → adapter — without pulling a component renderer into 1a to
render nothing. `renderHook` becomes the natural surface when STEP-4 and STEP-5 own real hooks;
adding the dependency then, rather than now, keeps the 1a dependency surface honest.

**T1 and T2 stay separate even though they share a runner.** Doc 11 §11.3 lists tests that are only
meaningful with middleware, error normalization, and the adapter wired together — §8.4's 401 scoping
above all. Testing the auth reducer alone would never catch a wrong password bouncing the user out
of the credentials screen, which is half of criterion **F2**. The distinction is not ceremony; it is
which tests are allowed to touch a store.

**T3 is deferred to 1b by doc 02**, not by this session. RISK-0001 and RISK-0011 carry it.

**T4 (Detox / Maestro) is declined for Phase 1, not deferred to 1b.** It needs a stable build and a
runner; there is no CI capable of building native artifacts until Bitrise (Phase 2, **OQ-05**), and
the sign-off artifact is a hand-installed release build smoked manually at every sync point (doc 09
§6.1). That manual smoke *is* the e2e layer for Phase 1 (§6). **Revisit trigger: Bitrise landing in
Phase 2.** Recorded as **RISK-0015**.

## 3. Coverage priorities

Not a percentage (§8). Each item below is either a launch criterion or a rule some doc states that
nothing else enforces — the ones that fail silently and are discovered on stage.

### 3.1 Auth path

| Must cover | Guards | Tier |
|---|---|---|
| Mock adapter credential compare, reading `DEMO_EMAIL` / `DEMO_PASSWORD` from `@env` — success and `INVALID_CREDENTIALS` | F2; doc 16 §1 ("auth decision is the mock adapter, not a screen `if`") | T1 |
| Adapter validates Bearer **presence and `exp`** on operations 2–5 → `UNAUTHORIZED` | Doc 11 §9.2 | T1 |
| **401 scoping** — `INVALID_CREDENTIALS` on `/auth/login` does *not* clear the session; `UNAUTHORIZED` on `/me` does | Doc 11 §8.4, **ADR-0017** | **T2** |
| Auth slice: login stores token + `expiresAt`; logout clears auth + user + content | Doc 16 §3 | T1 |
| **Persist whitelist is auth-only** — the RTK Query cache is absent from the persisted blob | ADR-0003, DF3, ADR-0020 | T1 |
| Boot gate holds the loader until all three of `/me` + HomeFeed + CW resolve | Doc 04 §6, §7 | T2 |

### 3.2 Storefront path

| Must cover | Guards | Tier |
|---|---|---|
| Home composition order: hero → `progress` → remaining HomeFeed containers | ADR-0006 | T1 |
| `nextCursor` exhaustion on **both** axes; the client trusts `nextCursor` over arithmetic on counts | Doc 11 §6.2 | T2 |
| Silent CW reload replaces **only** the `progress` container; hero and other rows untouched | Doc 04 §6 | T2 |
| Unknown `Container.variant` drops the row and emits `console.warn`; does **not** throw, does **not** fall back to a default layout | Doc 11 §4.4 | T1 |

### 3.3 API module

| Must cover | Guards | Tier |
|---|---|---|
| Both transports normalize to one `ApiError { code, status, message }` | Doc 11 §8.3 | T1 |
| Failure injection is reachable from tests only — no runtime path constructs it | Doc 09 §6.2 | T1 |
| Default adapter latency is nonzero and within doc 05's **400–600 ms** band | DF2 — see §9 | T1 |
| `remainingMinutes` is composed through i18n, never a hardcoded phrase | DF8, doc 04 §1.3 | T1 |

### 3.4 Theme

| Must cover | Guards | Tier |
|---|---|---|
| **Token parity** — the dark and light surface-mode token sets have identical key structure | **Criterion A1** | T1 |

Criterion A1 (a second theme re-skins both surface modes without touching a component) is otherwise
verified only by eye at the Tue-AM sync point, and its real failure mode is a token added to one
mode and forgotten in the other. A ten-line structural test makes the mechanical half mechanical;
the visual half stays in the smoke sequence (§6).

### 3.5 Explicitly not worth testing heavily

| Not covered | Why |
|---|---|
| Component rendering | 1b by decision (doc 02); RISK-0001 |
| Demo fixture *content* | Data, not logic. T0 already proves it conforms to `Container[]` / `Card[]`; asserting a card's title is testing a constant. The structural exception is §4.1's invariant test |
| Third-party behavior — axios, redux-persist, NetInfo | Test *our* interceptor and *our* whitelist, not their libraries |
| Snapshot tests of Emotion styled output | High churn, near-zero signal, and they would fight the token work rather than protect it |
| Navigation config, the analytics `console.log` stub, `ComingSoon` placeholders | No logic to protect |
| Criterion **A2** (zero hardcoded design values) | A **lint** rule, not a test (§7) |

**On the ~80% guide.** Carried as a steer, not a target. The list above is roughly what 80% of
`src/api/`, `src/features/*/state/`, and `src/features/*/hooks/` looks like anyway — but `src/app/`
and `src/shared/ui/` sit far below that in 1a *by design*, and chasing the aggregate would mean
writing the snapshot tests §3.5 just declined.

## 4. Test data & isolation

There is no database (doc 04 §3), so "a fresh schema per run" has no referent. What this project has
to isolate is different, and four of the five items below are obligations on the **foundation** and
**contract & mock API** STEPs (§11) rather than things tests can arrange for themselves.

### 4.1 Two fixture sets, and they are not the same thing

| Set | Lives | Used by | Rule |
|---|---|---|---|
| **Demo fixtures** | `src/api/mocks/fixtures/` | The app, in **both** build configurations (doc 09 §5) | Hero + 15 containers; page sizes per doc 11 §6.3. Typed `Container[]` / `Card[]`, never `any` (doc 11 §11.2) |
| **Test factories** | `src/api/mocks/` — colocated with the module that owns the type | Tests only | `makeCard(overrides?)`, `makeContainer(overrides?)`, `makePage(overrides?)`. Minimal objects, everything overridable |

**Tests assert against factories, not demo fixtures.** A test asserting `containers[3].name ===
'Series'` goes red the moment someone swaps a row for the demo — and under Sunday-night deadline
pressure that is how a genuinely broken test gets deleted instead of fixed.

**Factories live with the module that owns the type.** `Container` / `Card` factories sit in
`src/api/mocks/` because the API module owns those wire types (doc 03 §2, §8.1). A feature needing a
factory for its own slice shape puts it in that feature. There is no shared top-level `test/`
bucket.

Two consequences:

- **A test-time import is not an import-rule violation.** A storefront `*.test.ts` importing
  `makeContainer` from `src/api/mocks/` looks like a features → API import. Doc 03 §8.2 governs
  **runtime** paths ("never `axios`/`fetch` from a screen or hook"); test files are out of its
  scope. Stated here so a reviewer does not flag it on Sunday.
- **Factories must not be reachable from the app entry.** They live under `src/`, so anything
  non-test importing one puts them in the demo binary. Convention: `*.factory.ts`, imported only
  from `*.test.ts`. `tsc` still type-checks them.

**The one exception — a fixture-invariant test.** A single test runs against the *real* demo
fixtures and asserts only the structural rules the docs state: the first HomeFeed page is exactly
one `hero` plus 15 others (doc 04 §1.4); no `progress` container ever appears in a HomeFeed response
(ADR-0006); every card carries the artwork ratios its container's variant needs (doc 04 §1.2). That
catches a malformed fixture before the demo does, without coupling anything to content.

### 4.2 A store factory, not a store singleton

The scaffold exports **`createStore()`**; the app shell calls it once, tests call it per test. If
the shell exports a module-level `store` and tests import it, state leaks between tests and
produces order-dependent failures — the classic "passes alone, fails in the suite." The mock adapter
follows the same rule: constructed per test, with no module-level mutable state carrying cursors or
injected-failure flags across tests.

### 4.3 Tests never read the real `.env`

Doc 09 §7.2 already establishes that `.env` travels machine to machine by hand and goes stale. If
the suite resolved `@env` against a developer's actual `.env`, Dev A's tests would pass and Dev B's
would fail for a reason unrelated to the code, on the weekend with no time to chase it.

> **Jest maps `@env` to a committed stub** with fixed values. The real `.env` is a **build** input,
> never a **test** input.

The adapter still reads its credentials from `@env`, so doc 16 §1's "not a screen `if`" seam is
unchanged — only the resolution target differs between build and test. This also makes §7's CI tier
possible at all: a runner can never have the gitignored `.env`.

### 4.4 Latency and time are injected

| Seam | Default | In tests |
|------|---------|----------|
| **Latency** | Doc 05's 400–600 ms | `0` |
| **Clock** (`now()`) | `Date.now` | Frozen |
| **Failure injection** | Off | Per-test |

All three are **constructor parameters of the adapter**, not runtime toggles, and none survives into
the release binary — doc 09 §6.2's boundary, extended from failure injection to all three seams.

Latency is zeroed because ~400–600 ms across the tests this strategy implies is minutes of dead
wall-clock per run, and a slow suite is one people stop running. Constructor injection beats fake
timers, which get awkward around promise scheduling. The clock is injected because an `exp` test
written against `Date.now()` passes today and fails later for no reason. Fixture UUIDs are **fixed
committed constants**, never generated.

## 5. Mocking strategy

This project inverts the usual question: the mock adapter is a production artifact and one of the
primary things under test (§1). So the rule is narrow.

**Exercised for real:** reducers, selectors, middleware, hooks, the store, the mock adapter, the
`ApiError` normalization — and **i18n with the real Spanish string table**. Stubbing i18n to echo
keys would make the `remainingMinutes` test (§3.3) vacuous, since the whole point is that the phrase
is composed rather than hardcoded.

**Mocked only at the native boundary** — the modules with no JS implementation under Jest:

| Module | Why | Shape |
|---|---|---|
| `react-native-encrypted-storage` | No native module in Jest | In-memory `getItem` / `setItem`, so §3.1's persist-whitelist test can inspect what was written |
| `@react-native-community/netinfo` | Same | Official mock; controllable connectivity state |
| `@env` | Babel-transform resolved | Committed stub (§4.3) |

**Not mocked, not needed in 1a:** React Navigation — there are no component tests until 1b.

**`console.warn` is spied, not silenced.** Doc 11 §4.4 requires the warn to be observable; a blanket
console silence in test setup would quietly make that assertion unfallible.

**No consumer-driven contract testing** (Pact et al.) — doc 11 §11.1: one process contains both
sides of the boundary. Revisit at **OQ-34**.

## 6. System / end-to-end testing

The multi-repo question ("where do cross-repo e2e tests live — a dedicated tests repo?") dissolves
here. `registries/repos.yml` will hold the docs hub, `prompts/`, and **one** application repo
(`quasar-disney-mobile-app` — OQ-18 closed). Everything the system does happens in a single React Native process against in-process
mocks. There is no second deployable for an integration suite to sit between.

**The system test is the manual smoke on a release build, and it already exists.**
`runbooks/release-deploy.md` Part 3 walks F1 → F2 → F3 plus the VoiceOver/TalkBack pass, and doc 09
§6.1 requires it at **every** sync point rather than once at setup. This doc points at that runbook
rather than restating it; a second copy of the sequence is exactly the drift doc 11 §12 warns about.

### 6.1 Two smoke artifacts, not one

Doc 02 defers a "formal QA smoke checklist" to 1b while doc 09 §6.1 requires smoking `release` from
the first sync point. Both are right — they are different artifacts:

| Artifact | Phase | Owner | Scope |
|---|---|---|---|
| **Release smoke** — `runbooks/release-deploy.md` Part 3 | **1a, now** | **Raul Angel** (OQ-28 closed) | The launch criteria: F1/F2/F3 + a11y spot-check + A1 |
| **Formal QA checklist** | **1b** | QA | Exhaustive per-screen / per-state pass against installable builds |

The second needs CI-produced builds and a QA owner, neither of which exists before Phase 2.

### 6.2 Criterion A1 joins the smoke sequence

Part 3 smokes F1–F3 but not **A1**, and doc 02 §8 puts theme-swap verification at the Tue-AM sync
point where nothing enumerates it. §3.4's token-parity test covers the structural half; the visual
half is one step — switch to the test theme, confirm both auth and storefront re-skin — and this
session adds it to the runbook.

### 6.3 A2–A6 are not smoke items

It is tempting to hand the demo device a checklist of all eleven launch criteria. Only F1–F3 and A1
belong there:

- **A2** (no hardcoded design values) and **A5** (no cross-feature imports) are **lint rules** (§7).
- **A3** (API swap) and **A4** (contract delivered) are architectural facts, verified by review.
- **A6** (tests green) is the suite itself.

## 7. CI gates

Doc 11 §11.4 originally recorded **"none until Bitrise."** This session amends that — see
**ADR-0018** — because it conflated two different things. Bitrise is blocked externally on accounts
and signing because it builds **native binaries**. T0/T1/T2 are plain Node: `tsc --noEmit` and
`jest` need no simulator, no Xcode, no certificates, and — thanks to §4.3 — no `.env`.

| Tier | Runner | What | Phase |
|---|---|---|---|
| **A — JS gate** | GitHub Actions, `ubuntu-latest`, every push + PR | `tsc --noEmit` · `jest` · `eslint` · `prettier --check` | **1a**, from the scaffold STEP |
| **B — Native build** | Bitrise | Native build, installable QA artifacts, release smoke | **Phase 2**, externally blocked (**OQ-05**) |

Stamped from `templates/ci/code-repo-ci.yml` into the app repo's `.github/workflows/ci.yml` when the
repo is scaffolded. The docs hub already runs `method-check.yml` (`scripts/check.sh`).

### 7.1 What blocks merge

| Gate | Enforces |
|---|---|
| `tsc --noEmit` | Doc 11 §11.2 — the contract test |
| `jest` (T1 + T2) | Criterion **A6** |
| ESLint `import/no-restricted-paths` | **DF5 / criterion A5** — no feature imports another feature; `shared/` never imports `features/` |
| ESLint restricted import of `axios` / `fetch` outside `src/api/` | **DF1** — doc 03 §8.2, made mechanical |
| ESLint restriction on color/spacing literals outside `src/shared/theme/` | **Criterion A2** — with the caveat below |
| `prettier --check` | Keeps two developers' diffs from colliding on formatting at the Sat-EOD merge |

The first four turn architecture criteria that were previously *review-enforced* into
*machine-enforced*. A5 and DF5 are the whole extractability argument (doc 02 DF5) and the seam that
makes the two-developer split work — worth more than a code-review habit under deadline pressure.

**The A2 rule is honest about its reach.** `no-restricted-syntax` against hex/`rgb()` literals and
raw `px` values outside `src/shared/theme/` catches the common case. It will **not** catch a color
computed at runtime. Shipping the 90% rule beats pretending review covers the rest — recorded as
**RISK-0017** so the gap is not mistaken for full automation.

### 7.2 What blocks the release build

Nothing new. Doc 09 §7.1 already requires cutting from a tagged trunk commit, so **trunk-green
(Tier A)** plus `runbooks/release-deploy.md`'s pre-flight (`.env` key completeness, clean device
state) and Part 3 smoke *are* the deploy gate. This section names them as such rather than inventing
machinery.

### 7.3 Not gated

| Not a gate | Why |
|---|---|
| `npm audit` | Deferred — **RISK-0010**; decided in the Bitrise STEP |
| Coverage thresholds | §8 — no numeric gate in 1a |
| OpenAPI / schema linting | No such artifact exists by decision (ADR-0016); doc 11 §11.4 calls linting it theatre |
| ShellCheck / shfmt | Tier A is a JS-only job; §10 |

**Speed budget: under two minutes.** With adapter latency at zero (§4.4) the suite is fast. A gate
slower than that is one people route around three days before a fixed date.

## 8. Coverage tooling and reporting

| Surface | Tool | Durable summary | Generated artifact | Threshold / gate |
|---|---|---|---|---|
| TypeScript (all of `src/`) | **Jest built-in coverage, `coverageProvider: 'babel'`** (Istanbul) | `reports/test-results/` — meaningful runs only (§8.2) | CI artifact; `coverage/` gitignored locally | **None in 1a** |
| API contract | `tsc --noEmit` | — | — | Blocks merge (§7.1) |
| Shell (`scripts/*.sh`) | None — `kcov` not warranted at this volume | — | — | — |

**Istanbul rather than V8.** The RN pipeline already runs everything through Babel
(`metro-react-native-babel-preset` plus `react-native-dotenv`, ADR-0015). V8 coverage maps native
coverage data back through those transforms and misattributes lines often enough to be annoying;
Istanbul instruments during a transform that is already happening. Marginally slower, materially
more trustworthy.

### 8.1 No numeric threshold in Phase 1a

Aggregate coverage will be structurally low **by decision**: doc 02 defers all UI/component tests to
1b, so `src/shared/ui/` and most of `src/app/` are untested on purpose. A global threshold is then
one of two bad things — high enough to be meaningful and it fails the build for an architectural
decision deliberately made, or low enough to pass and it certifies nothing.

**§3's must-cover list is a strictly better enforcement mechanism than a percentage.** It is
specific, reviewable in a PR, and names the exact behaviors — 401 scoping, two-axis exhaustion,
persist whitelist — that a number would happily let someone skip in favor of something easier to
reach.

Instead:

- **Coverage is reported on every CI run** — the text summary in the job log, visible on every PR.
  Trend and visibility, which is what coverage is actually good for.
- **Per-path thresholds are a named 1b action**, not a 1a gate: `src/api/**` and
  `src/features/*/state/**` at ~80% once the suite has settled and UI tests make the global figure
  honest. Introducing them on Saturday would fail a foundation PR that scaffolds `src/api/` with
  tests not yet written — incomplete but correct. Recorded as **RISK-0016**.

**No changed-lines threshold and no coverage service.** Codecov or Coveralls would be the first
external vendor in a project that has deliberately taken none (doc 03 §7, doc 10 §5.1).

### 8.2 Durable summaries

Generated HTML stays a CI artifact and gitignored local output — no committed multi-file trees
(`reports/test-results/README.md`). A Markdown summary is written to **`reports/test-results/`**,
started from `templates/reports/test-results/test-results-summary-template.md`, only for runs that
carry weight:

- the **2026-08-18 sign-off build**,
- each **Check-in STEP** (`runbooks/check-in.md` already runs a full suite),
- any **incident** follow-up or **security review** that turns on test evidence.

Not per-PR and not per-merge — that would turn the docs hub into a log file.

## 9. Performance & load testing

**None — declined, with a trigger.** Load testing measures a service under concurrency. There is no
service (doc 08 §1), and every "request" is an in-process Promise resolving a committed fixture
after a delay this project chose. Doc 05 decision 3 already forecloses a perf program for the demo
and doc 15 §8 declines numeric SLOs; this session confirms rather than reopens.

**Revisit trigger:** Phase 3, when `API_BASE_URL` addresses a real host. Worth recording *whose* job
it is: **load-testing the backend belongs to the backend team**, not to this client. The mobile-side
equivalent is **network-condition testing** — slow and flaky links, timeouts, retry behavior — which
needs a real network to mean anything.

**The performance properties that matter here are guarded by correctness tests**, not by a perf
tier:

| Doc 15 §8 "what would blow it" | Guarded by |
|---|---|
| Persisting the RTK Query cache | §3.1 persist-whitelist test |
| Loading every tile in every row on first paint | §3.2 two-axis pagination tests |
| Shipping the debug bundle to the demo device | `runbooks/release-deploy.md` pre-flight |
| Unbounded image decode | Not testable in 1a — virtualization is a component concern; lands with 1b's UI tests |

**One test earns its place here:** §3.3's assertion that the adapter's **default** latency is nonzero
and inside doc 05's 400–600 ms band. DF2's whole argument is that loading and error states are real
code paths rather than decorative, and the most plausible way that silently dies is someone zeroing
the default while chasing a slow suite — having seen §4.4 set latency to zero *in tests*. The test
pins the distinction between the two.

## 10. Coding standards

Implementation language from doc 03: **TypeScript, only.** `coding-standards/` is reconciled to
that list.

| File | Disposition | Why |
|---|---|---|
| [`typescript.md`](../coding-standards/typescript.md) | **Kept, amended** | The one implementation language |
| [`shell.md`](../coding-standards/shell.md) | **Kept, amended** | The hub ships `scripts/*.sh` and root `doctor.sh`; doc 09 §7.2 contemplates a `.env` completeness script |
| [`api.md`](../coding-standards/api.md) | **Kept, substantially amended** | Doc 11's contract is REST-shaped and is *delivered* to a backend team (criterion A4); boundary B-B becomes real HTTP at Phase 3 |
| `sql.md` | **Pruned** | Doc 04 §3: no relational DB, no SQLite/MMKV, no server DB |
| `python.md` `go.md` `rust.md` `dart.md` `java.md` `csharp.md` | **Pruned** | Not used at any phase in the roadmap |

### 10.1 `typescript.md` — what changed

The shipped default is a generic **Node** TypeScript standard, and roughly a third of it contradicted
decisions this project had already made. Left unamended it would have described a house style the
project's own architecture violates.

| Shipped default | This project | Source |
|---|---|---|
| "Structured logger (pino/Winston); never `console.log` in shipped code" | **`console` is the sanctioned mechanism.** No logging library. Doc 10 §2.3's never-log list is binding | Doc 10; DF9; doc 11 §4.4 requires a `console.warn`; doc 09 §4.3 strips `console.*` from release |
| Runtime "Node 20+" | **Hermes** is the app runtime; **Node 20** is the tooling/CI runtime | Doc 15 §8 |
| `node:worker_threads` for CPU work | Does not exist in React Native | Doc 15 §1 |
| ESM via `"type": "module"`; output to `dist/` | ESM **syntax** yes; no `package.json` type flag, no `dist/` — Metro bundles, `tsc` runs `--noEmit` | §7.1 |
| "Vitest (or Jest)" | **Jest**, React Native preset | §2 |
| Generic `Error` subclass hierarchy | **`ApiError { code, status, message }`** — every transport normalizes into it; features switch on `code` | Doc 11 §8.3 |

Two forks the file leaves open are **settled**, because two developers working in parallel on
disjoint branches (doc 02 §9) turn an unsettled convention into merge noise at the Sat-EOD sync
point:

- **Filenames:** `PascalCase.tsx` for React components (matching doc 07's atomic inventory),
  `camelCase.ts` for everything else — slices, hooks, adapters, factories.
- **Test location:** **colocated `*.test.ts`** next to the source file, not `__tests__/` —
  consistent with §4.1's colocation rule for factories.

**`noUncheckedIndexedAccess` stays off** for 1a. It is genuinely safer, but carousel code is full of
`resources[0]`-shaped access and it would add narrowing friction on every row across a 3.5-day
build. `strict: true` remains required. Revisit at 1b.

**"No `any`" is strengthened** with a cross-reference: doc 11 §11.2 makes typed fixtures the
condition on which the entire type-as-contract strategy rests. It is not generic style advice here.

### 10.2 `api.md` — what changed

The shipped default is an opinionated REST house style, and doc 11 deliberately went the other way
on six of its rules, each with recorded reasoning. `api.md` is now thinner and defers to doc 11 as
the contract of record.

| Shipped default | Doc 11 | Revisit |
|---|---|---|
| `snake_case` fields | **camelCase** (§6.3) — no serialization layer; a mapper buys nothing | — |
| **RFC 9457 Problem Details** | **Declined** (§8.1) — `type` needs a host we do not have; its prose fields fight i18n | §14 item 4 |
| Correlation / request ID | **Declined**, no reserved header name (§10; doc 10 §2.4) | §14 item 2 |
| URI versioning `/v1/` | **Declined** (§4.1) — absorbed by `API_BASE_URL` | §14 item 5 |
| `Idempotency-Key` | **None** (§6.3) — login is the only non-GET and is naturally idempotent | — |
| Spectral OpenAPI linting | **No OpenAPI artifact** (ADR-0016) | §14 item 1 |
| `limit` default 25 / max 100 | **16** on `/home-feed`; **10** on CW and `resources` (§6.3) | — |
| Cursor via `Link` header *or* envelope | **Envelope** `{ data, nextCursor }`, two levels (§6.1–§6.2) | — |
| Rate limiting + `429` | Parked — RISK-0009 | §14 item 7 |

Three deliberate deviations are now **recorded in `api.md`** so nobody "fixes" them later:

- **`POST /auth/login` is a verb path** and **`GET /me` is a singleton**, both breaching "nouns,
  plural collections." Both are near-universal auth idioms and both are already in the contract
  handed to the backend team.
- **Enum value casing splits by domain, deliberately:** domain enum values follow field casing
  (`standardPortrait`, `standardLandscape`); **error codes are `UPPER_SNAKE_CASE`**
  (`INVALID_CREDENTIALS`). Unstated, the mock and a future backend would disagree on the first new
  value either one adds.
- **`null` vs omitted is principled:** `null` for a field that always exists but may be empty
  (`rating`); **omitted** for a field that does not apply to that variant (`progress?`).

### 10.3 `shell.md` — what changed

- **Shebang settled: `#!/usr/bin/env bash`** — the convention all seven existing scripts already use,
  and correct on macOS where `/bin/bash` is still 3.2.
- **Docker-entrypoint framing dropped** — doc 08 §1 hosts nothing and there are no containers.
- **The "past ~100 lines, rewrite in Python/Go" escape hatch retargeted** — neither language is in
  this stack; past that size it belongs in TypeScript under the Node tooling runtime, or it should
  be split.
- **ShellCheck / shfmt are not a Phase-1 CI gate** and the file now says so. Tier A is a JS-only job
  (§7); they stay a local/pre-commit recommendation. Claiming they are "pinned in CI" when they are
  not is how a standards doc becomes untrustworthy.

## 11. Scaffold obligations

Doc 11 §3.1 established the pattern: a session that depends on structure the scaffold creates states
that dependency as an obligation rather than hoping it is remembered. Five items here are of that
kind — each is cheap on Saturday and expensive on Sunday, because retrofitting any of them means
touching every test already written.

**The foundation STEP must:**

1. Export **`createStore()`** — a factory, not a module-level singleton (§4.2).
2. Configure Jest to map **`@env` to a committed stub** (§4.3), and to mock
   `react-native-encrypted-storage` and `@react-native-community/netinfo` (§5).
3. Stamp **`templates/ci/code-repo-ci.yml`** into `.github/workflows/ci.yml` with the Tier A
   commands (§7).
4. Configure **ESLint** with the four rules in §7.1, and Prettier.

**The contract & mock API STEP must:**

5. Build the adapter with **latency, clock, and failure injection as constructor parameters**
   (§4.4), and ship the **test factories** alongside the demo fixtures (§4.1).

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Test tiers | **T0 type check · T1 unit · T2 integration**; T3 component in 1b; **T4 device e2e declined** | Doc 02 A6 already fixes the shape; the open question was where the lines fall | Automated device e2e on 18 Aug (**RISK-0015**) |
| 2 | T1 / T2 split | Kept separate despite sharing a runner | Doc 11 §11.3's tests are only meaningful with store + middleware + adapter wired; F2's 401 scoping is invisible to a reducer test | A single flat "unit" tier |
| 3 | Coverage priorities | **A named must-cover list** (§3), not a percentage | Specific and reviewable; a number lets you skip the hard behaviors for reachable ones | A single coverage figure as the quality signal |
| 4 | Two tests beyond what the docs required | **Persist whitelist** and **theme token parity** | ADR-0003 and criterion A1 were otherwise enforced by eye alone | — |
| 5 | Fixture strategy | **Demo fixtures ≠ test factories**; factories colocated with the module owning the type | Asserting on demo content makes unrelated tests red when artwork changes | Reusing the shipped fixtures as test input |
| 6 | Fixture invariants | **One** test asserts structural rules against the real demo fixtures | Catches a malformed fixture without coupling to content | — |
| 7 | Store construction | **`createStore()` factory**, never a shared singleton | Module-level state leaks between tests and produces order-dependent failures | Importing the app's store into tests |
| 8 | `@env` in tests | **Committed stub**, never the real `.env` | `.env` travels by hand and goes stale (doc 09 §7.2); it also cannot reach a CI runner | Tests reading real credential values |
| 9 | Adapter seams | **Latency, clock, failure injection** are constructor parameters | A 400–600 ms suite is one people stop running; `Date.now()` makes `exp` tests time bombs | Runtime toggles; a debug menu in the binary (doc 09 §6.2) |
| 10 | Mocking | Mock **only** at the native boundary; the mock adapter and i18n run for real | The adapter is a production artifact under test, not a double | Stubbing i18n (would make the DF8 test vacuous) |
| 11 | `console.warn` | **Spied, not silenced** | Doc 11 §4.4 requires the warn to be observable | A blanket console silence in test setup |
| 12 | System / e2e | The **manual release smoke** is the system test; the runbook owns it | One process, one repo — nothing for an integration suite to sit between | Automated e2e until Bitrise (Phase 2) |
| 13 | Smoke artifacts | **Two:** 1a release smoke (exists) vs 1b formal QA checklist | Resolves doc 02 / doc 09 §6.1's apparent conflict — they are different artifacts | — |
| 14 | Criterion A1 | Structural half is a **test**; visual half joins the **runbook** smoke | A1 was verified by eye at a sync point nothing enumerated | — |
| 15 | CI gates | **Two tiers** — JS gate now (GitHub Actions), native build at Bitrise (**ADR-0018**) | Bitrise is blocked on signing, which the JS suite does not need. §4.3 is what makes a runner possible | Amends doc 11 §11.4's "none until Bitrise" |
| 16 | Lint as architecture enforcement | **DF5/A5, DF1, A2** become ESLint rules | Machine-enforced beats a convention two devs work against separately all weekend | A2's rule is pattern-matching (**RISK-0017**) |
| 17 | Coverage tooling | **Jest + Istanbul** (`coverageProvider: 'babel'`) | V8 misattributes lines through the RN Babel chain | Marginally slower instrumentation |
| 18 | Coverage gate | **None in 1a**; reported on every run; per-path thresholds at 1b | UI tests are deferred *by decision*, so any global number is meaningless or punitive | **RISK-0016**; no changed-lines gate |
| 19 | Coverage service | **None** — no Codecov/Coveralls | Would be the first external vendor in a project that has taken none | Hosted trend graphs and PR annotations |
| 20 | Durable summaries | `reports/test-results/` for **sign-off, check-ins, incidents, security reviews** only | Per-PR summaries would turn the docs hub into a log file | — |
| 21 | Performance / load testing | **Declined**, not deferred | No service to load; every request is a Promise over a fixture | Confirms doc 05 decision 3 and doc 15 §8 |
| 22 | Latency default test | Assert the default is nonzero and in doc 05's band | The likeliest way DF2 dies is someone zeroing it after seeing §4.4 | — |
| 23 | Coding standards | Keep **`typescript.md`**, **`shell.md`**, **`api.md`**; prune the other seven | Doc 03's stack is TypeScript; SQL has no referent (doc 04 §3) | — |
| 24 | `typescript.md` logging rule | **Reversed** — `console` is sanctioned; no pino/Winston | Doc 10 declined the telemetry stack; DF9 *is* `console.log`; doc 11 §4.4 *requires* `console.warn` | A structured logger in Phase 1 |
| 25 | `typescript.md` open forks | **Filenames and test location settled** | Two parallel branches merging Sat EOD; an unsettled convention is merge noise | `__tests__/` directories |
| 26 | `noUncheckedIndexedAccess` | **Off** for 1a; `strict` stays on | Carousel code is `resources[0]`-shaped throughout; friction on every row before a fixed date | Revisit at 1b |
| 27 | `api.md` | **Thinned**; defers to doc 11 and records three deliberate deviations | A house style the project's own contract violates is worse than none | — |
| 28 | Scaffold obligations | **§11 collects five**, in doc 11 §3.1's pattern | Each is cheap Saturday and expensive Sunday | Requires the foundation STEP to read this doc |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-35 | When Bitrise lands in Phase 2, does it **subsume** the Tier A GitHub Actions job or do both run? Doc 09 decision 3 calls CI "a runner"; there would then be two | Mobile | Phase 2 Bitrise STEP (**OQ-05**) |
| ~~OQ-36~~ | ~~Who owns fixing a red trunk during the Sat–Mon parallel window?~~ **Resolved (planning session):** the author of the PR that went red fixes it before anything else merges. If trunk is red after combining both STEPs, the later merger owns the fix. Fallback: **Raul Angel** (STEP-2 CI owner). | — | closed |

Carried forward, unchanged by this session: **OQ-05** (Bitrise setup — §7 Tier B), **OQ-34** (backend accepts the contract — §5's revisit for
consumer-driven contract testing).

**Closed by the planning session:** **OQ-12** (Dev A = Raul Angel, Dev B = Andres Montoya), **OQ-18** (`quasar-disney-mobile-app`), **OQ-28** (Raul Angel owns the sign-off binary), **OQ-36** (red-trunk owner rule above).

**Opened as accepted risks:** **RISK-0015** (no automated e2e), **RISK-0016** (coverage reported but
not gated), **RISK-0017** (criterion A2's lint rule is pattern-matching).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.12 | Initial draft from the test-strategy session. Four tiers set (device e2e declined); must-cover list named, adding the persist-whitelist and theme token-parity tests; test factories separated from demo fixtures and colocated; `@env`, latency, clock, and failure injection fixed as test seams; two-tier CI adopted (**ADR-0018**, amending doc 11 §11.4) with DF5/DF1/A2 as lint gates; coverage reported but not gated; load testing declined; `coding-standards/` reconciled to TypeScript + shell + api, seven files pruned. **ADR-0017** relocates doc 11 §8.4's 401 policy above the transport. Opened OQ-35, OQ-36 and RISK-0015/0016/0017. |
| v0.1.1 | 2026-08-17 | STEP-1.14 | T2 is store + `baseApi` + `axios-mock-adapter`. Persist whitelist excludes the RTK Query cache. Emotion, not styled-components snapshots (ADR-0019, ADR-0020). |
| v0.1.2 | 2026-08-17 | planning session | Closed OQ-36 (red-trunk owner). Recorded OQ-12 / OQ-18 / OQ-28 closures. |
| v0.1.3 | 2026-08-18 | STEP-3.5 | **T2 exists.** §2's T2 row is corrected: the layer landed as store-driven suites in `src/api/integration/` with endpoints injected test-locally, and **RNTL is not yet a dependency** — `renderHook` arrives with the first feature-owned hook in STEP-4/5. §3.1's 401-scoping, §3.2's two-axis cursor, §3.3's `ApiError` and injected-failure rows are covered; §3.1's **boot gate** row stays open and belongs to the shell (STEP-6), and its persist-whitelist row is covered by `createStore.test.ts` (T1, STEP-2.3). No change to the tiers, the gates, or the coverage posture. |
