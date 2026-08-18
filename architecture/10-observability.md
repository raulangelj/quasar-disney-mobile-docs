# Doc 10 — Observability

**Version:** v0.1.1
**Status:** Draft
**Coverage:** full for Phase 1. Metrics, tracing, health checks, dashboards, and alerting are
**consciously declined** with named revisit triggers (§3, §4, §6) rather than left unenumerated.
**Last updated:** 2026-08-17 (STEP-1.10)
**Audience:** Mobile developers, QA, eng leadership

> What this app logs, what it deliberately does not measure, and the one failure mode worth
> building for — a JavaScript exception during the 2026-08-18 demo.

## Table of Contents

1. [Observability without a collector](#1-observability-without-a-collector)
2. [Logging](#2-logging)
3. [Metrics — none](#3-metrics--none)
4. [Tracing & health checks — none](#4-tracing--health-checks--none)
5. [Error tracking — one boundary, no vendor](#5-error-tracking--one-boundary-no-vendor)
6. [Dashboards & alerts — none](#6-dashboards--alerts--none)
7. [Tooling & retention](#7-tooling--retention)
8. [The analytics stub (restated, not reopened)](#8-the-analytics-stub-restated-not-reopened)
9. [What this accepts](#9-what-this-accepts)

---

## 1. Observability without a collector

The standard three pillars — logs, metrics, traces — all assume the same thing: somewhere to
*send* the signal, and someone who will look at it later. Doc 08 §1 is unambiguous that neither
exists here. There is no server, no collector, no hosting tier, no on-call rotation, and doc 05
§1 puts the load at **fewer than ten concurrent sessions**, usually one device at a time, with
all I/O served by in-process mocks.

Reproducing the standard model against that would produce instrumentation nobody reads, emitted
to a sink nobody runs, measuring a system with no users. So this session does not scale the
standard model down. It asks the two questions this project actually has:

1. **Can a developer diagnose a failure at the console during development?**
2. **Does an unexpected exception during the demo destroy it, or degrade into something
   recoverable?**

Question 1 is answered by §2 with existing tools and no new code. Question 2 is answered by §5
and is the only thing this session adds to the build. Everything else is declined in place, with
the trigger that would reopen it — the same posture doc 09 took to the `staging` tier.

## 2. Logging

### 2.1 No logger module

**Logging is bare `console.*` at call sites.** No `shared/logging/` module, no level enum, no
structured envelope, no log-event registry.

A wrapper module earns its place when call sites must be migrated to a different sink without
being rewritten. Phase 1 has no second sink and — as a POC that is not scheduled to go further
(doc 01: internal demo, not a store release) — no planned migration to one. The wrapper would be
indirection protecting against a future that is explicitly out of scope.

| Level | Use | Present in `release`? |
|-------|-----|----------------------|
| `console.log` / `console.info` | Development trace: mock latency, middleware transitions, analytics stub events | No — stripped |
| `console.warn` | Recoverable oddity: unexpected mock state, missing optional field | No — stripped |
| `console.error` | Caught exception, failed request the user was not shown an error for | No — stripped |

Gate anything genuinely noisy behind **`__DEV__`** (doc 09 §4.3), so it is provably absent from
the binary rather than merely quiet.

### 2.2 The release binary is silent, and that is the decision

Doc 09 §6 already records that `console.*` is removed from the `release` configuration. Nothing
here preserves `warn`/`error` through the strip.

The consequence is worth stating plainly rather than discovering it: **a failure in the sign-off
binary produces no console output**, even with the device attached to Xcode or `logcat`. The
compensating controls are already in place and were not added for this reason:

- The `release` configuration is **built and smoked at every sync point** (doc 09 §6.1), so
  release-only breakage is found before the 18th rather than on it.
- A render exception degrades into a recoverable screen instead of a white one (§5).
- A binary that misbehaves is rolled back by reinstalling the previous known-good artifact —
  **~1 minute** (doc 08 §7).

If diagnosis is genuinely needed on a device, the answer is to install the `development`
configuration on it, not to add a logging path to `release`.

### 2.3 Never-log list

Doc 06 §5/§7 and doc 15 §6 already forbid logging the password, the JWT, and the full `/me`
response. Doc 06 nominated an *API interceptor* as the enforcement mechanism. This session
satisfies that with a simpler and stricter rule:

> **Do not log request or response bodies at all.** Not redacted, not truncated — not logged.

Redaction is a mechanism that has to be maintained: it needs a key list, and it silently stops
working the moment a new field is added. Not logging bodies has nothing to maintain and no
failure mode, and it costs nothing in a codebase whose entire I/O surface is five mock functions.

| Never logged | Why |
|--------------|-----|
| `password` | Credential (doc 06 §2) |
| JWT / `Authorization` header value | Session token (doc 15 §6: "tokens never logged") |
| `/me` response body, `userName` | User data — the shapes are treated as production even though the person is fake (doc 06 §1) |
| Container / Card payloads | No value at the console, and it trains the wrong habit for Phase 3 |

**Fine to log:** event names, HTTP-ish status codes, error codes and messages, durations,
container and card *counts*, pagination cursors' presence (not opaque values).

### 2.4 No correlation IDs

Declined. Phase 1 has one process, one device, and zero network hops — a request ID would
correlate a call with itself.

**Revisit trigger: Phase 3 backend integration.** At that point the ID has to be minted in the
axios interceptor *and* echoed by the backend, so it is a joint decision with whoever implements
the API, not something to guess at now. It joins the contract conversation in doc 11 (OQ-10).

## 3. Metrics — none

**No counters, no timers, no golden signals, no domain metrics.**

Doc 05 §2 already removed numeric SLOs from the 18 Aug gate: no FPS target, no binary-size
target, no cold-start SLO, no p99. Its targets are **perceived UX** — mock latency 400–600 ms,
first paint under roughly two seconds — and the stated remedy for a device that hitches is *"use
a release build; do not open a performance program."* Instrumenting numbers that no threshold
depends on produces a dashboard-shaped object with no dashboard.

Domain metrics are equally empty: there is one shared demo account (RISK-0008), no signup funnel,
no playback, and no retention to measure. The analytics hook stub (§8) is where such events
*would* originate, and in Phase 1 it originates them to `console.log`.

**Revisit trigger:** real users, or a Phase-3 backend that can receive an event stream.

## 4. Tracing & health checks — none

**Distributed tracing: none.** It requires a request crossing process boundaries. Doc 03 §2 is a
modular monolith in a single React Native process, and doc 05 §5 makes the mocks Promises with
artificial latency. There is no hop to trace.

**Health checks: not applicable.** Liveness and readiness endpoints exist so runtime
infrastructure can probe a service and act on the answer. Doc 08 §1 deploys nothing — no service,
no orchestrator, no load balancer, no probe. The app's equivalent of "is it up" is that someone
is holding it.

The closest real analogue already exists and is not a health check: the **NetInfo connectivity
gate** (ADR-0004, ADR-0014) keys on interface state and is a *user-facing* overlay, not a
monitoring signal.

**Revisit trigger for both:** Phase 3, when a real host is on the other end of the API module.

## 5. Error tracking — one boundary, no vendor

### 5.1 No error-tracking vendor

**No Sentry, Crashlytics, Bugsnag, or Firebase.** Doc 06 §6 already carries this as a standing
rule ("Do not add Firebase/Sentry 'for security'"), and doc 03 §7's hard-dependency list is
**unchanged** by this session — the boundary below is React, not a package.

The rationale is the same as the rest of this doc: a crash reporter's value is aggregate crash
volume across a user base, and there is no user base. For a demo held by two developers, the
crash report is *the person watching the screen*.

### 5.2 One error boundary at the shell root

This is the one thing 1.10 adds to the build. Today, an unhandled exception in any render path
unmounts the React tree and leaves a **white screen with no route back** — during a demo whose
sign-off criterion is visual fidelity, in a binary with no console output (§2.2). The fix is
roughly twenty lines and zero dependencies.

| Property | Decision |
|----------|----------|
| **Where** | **One boundary at the app-shell root** — the shell is the composition root (doc 03 §2). Mounted *inside* the theme provider so the fallback can use theme tokens and Shared atoms |
| **Scope** | Root only. Not per-feature, not per-screen — the failure being prevented is the white screen, and more boundaries is more surface to build and test for no POC benefit |
| **Fallback UI** | Minimal, from doc 07's existing atoms: a short message and a single retry control. No stack trace, no error code, nothing that reads as a debug affordance on stage (**OQ-33** designs it) |
| **Retry** | Remounts the subtree via a reset key. Auth state survives — it is in the persisted slice (ADR-0003), not React state — so a retry lands the user back where they were, not at Welcome |
| **Logging** | `console.error` the caught error and component stack. Development only, by §2.2's strip |
| **Ships in** | **Both configurations.** Unlike the debug affordances doc 09 §6.2 keeps out of `release`, this is product behavior, not a test seam |

### 5.3 What an error boundary does not catch

Error boundaries are routinely over-trusted, so the gaps are recorded here rather than assumed.
Each one is already handled by an existing decision:

| Not caught by the boundary | Already handled by |
|---------------------------|--------------------|
| Async rejections — middleware and API-module failures | Feature-level error states: inline credentials error (doc 02 F2), storefront error/empty state. "Request failed" is a feature concern (doc 15 §2) |
| Event-handler exceptions | Same as above; the card-tap handler's only action is an `Alert` (doc 03 §6) |
| Loss of connectivity | Shell NetInfo overlay — a shell concern, deliberately *not* conflated with request failure (ADR-0004, ADR-0014) |
| A corrupt persisted auth blob | Purges rather than crashes (doc 04, doc 08 §6) |
| Native crashes | No JS boundary helps. Mitigation is doc 08 §7: keep the previous known-good binary, reinstall in ~1 minute |

## 6. Dashboards & alerts — none

**No dashboards, no alert rules, no thresholds, no notification targets, no on-call.**

Alerting exists to tell an absent human that users are feeling something. In Phase 1 there is no
collector to draw a dashboard from (§3), no production to page about (doc 08 §1), and the
audience is in the room with the device. Writing an alert table here would mean inventing both
the metric and the person who receives it.

The operative "alerting" mechanism for 18 Aug is the pre-flight sequence that already exists in
`runbooks/release-deploy.md` and doc 08 §6 — clean-state install verified at the welcome screen,
`.env` key completeness, live network, spare binary on disk. That is a checklist, not monitoring,
and it is the correct instrument for a one-off demo on a known date.

**Revisit trigger:** the first time anyone other than the two developers depends on a running
build.

## 7. Tooling & retention

| Signal | Sink | Retention |
|--------|------|-----------|
| Console output, `development` | Metro terminal, Xcode console, `adb logcat` | **Session lifetime.** Nothing written to disk by the app |
| Console output, `release` | **None** — stripped at build (doc 09 §6) | N/A |
| Analytics stub events | `console.log`, development only | **None** |
| Crashes / exceptions | Not collected. Surfaced in-app by §5.2 | **None** |
| Metrics, traces | Do not exist | N/A |

**No log files on device, no local log store, no export path, no vendor account, no cost** —
consistent with doc 08's $0 posture. Because nothing is written to disk, doc 04 §5's retention
and deletion table needs no new row: there is no observability data at rest to retain or delete,
and uninstalling the app removes everything by construction.

## 8. The analytics stub (restated, not reopened)

Docs 01, 03 §4, 04 §5, and 15 §3 already commit to an **analytics hook stub** in
`shared/analytics/`: the hook shape is designed, the implementation is `console.log`, there is no
vendor, and **no PII appears in stub payloads**. This session changes none of it.

What it adds is the connection: the analytics stub *is* Phase 1's product-metrics story, and its
payload rule is the same never-log list as §2.3. When a real analytics sink arrives in a later
phase, it lands **behind that hook** — call sites do not change. That is the same swap-behind-a-
boundary shape as the API module (ADR-0002) and the IdP (ADR-0009), and it is why the stub is
worth having even though nothing consumes it today.

## 9. What this accepts

Stated once, honestly: **this app has no telemetry.** If the sign-off binary misbehaves in a way
that is not visible on screen, there is no signal — no log, no metric, no crash report — and
diagnosis means reproducing it on a `development` build.

That is the correct trade for an internal POC held by its own authors on a known date, and it
would be negligent for a product with users. The line between those two situations is recorded as
**RISK-0014**, whose revisit trigger is the moment anyone outside the two developers depends on a
build, or any real user data exists.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Observability model | **Diagnose at the dev console; survive a demo-time exception.** The three-pillar model is declined wholesale, not scaled down | No collector, no on-call, <10 sessions, one process per device (doc 05 §1, doc 08 §1) | Any signal that outlives the session it was produced in |
| 2 | Logger | **None** — bare `console.*`, `__DEV__`-gated where noisy | A wrapper protects a sink migration that this POC has explicitly ruled out (doc 01) | Structured logs; a log-event registry; cheap re-pointing of call sites at a real sink later |
| 3 | Release logging | **Silent binary** — doc 09 §6's strip stands; `warn`/`error` are not preserved | Compensated by release smoke at every sync point, §5's boundary, and a ~1-min rollback | On-device diagnosis of the sign-off binary; install `development` instead |
| 4 | Never-log enforcement | **Do not log request/response bodies at all** — stricter and simpler than redaction | Redaction needs a maintained key list and fails silently when a field is added | Body-level debugging at the console |
| 5 | Correlation IDs | **None.** Trigger: Phase 3 | One process, zero hops — it would correlate a call with itself; the header is a joint decision with the backend | Retrofit cost at Phase 3, accepted knowingly (doc 11 / OQ-10) |
| 6 | Metrics | **None** — no golden signals, no domain metrics | Doc 05 §2 already dropped numeric SLOs; no threshold depends on a number, and there is no funnel to measure | Any quantitative claim about the demo's performance |
| 7 | Tracing | **None** | Single process; mocks are Promises | — |
| 8 | Health checks | **Not applicable** — nothing is deployed to probe | Doc 08 §1. The NetInfo gate is user-facing, not a monitoring signal | Confusing the connectivity overlay with a health probe |
| 9 | Error-tracking vendor | **None** — reaffirms doc 06 §6; doc 03 §7 dependency list **unchanged** | A crash reporter's value is aggregate volume across a user base that does not exist | Post-hoc crash analysis; a Phase-2 addition either way |
| 10 | Error boundary | **One at the shell root**, inside the theme provider, shipping in **both** configurations | A render exception currently means a white screen with no route back, in a binary with no console — ~20 lines, zero dependencies | Per-feature boundaries; granular recovery. Fallback design is **OQ-33** |
| 11 | Dashboards & alerts | **None** | Nothing to draw from, nobody to page, audience is in the room. The pre-flight checklist is the right instrument | Treating a checklist as monitoring beyond this demo |
| 12 | Tooling & retention | **No tooling, no retention, $0.** Console output dies with the session; nothing on disk | Consistent with doc 08's cost posture; leaves no observability data at rest | Doc 04 §5 needs no new row — nothing is retained to delete |
| 13 | Analytics stub | **Unchanged** from docs 01/03/04/15; recorded as Phase 1's product-metrics story | Swap-behind-the-hook mirrors ADR-0002 and ADR-0009 | Reopening a decision four docs already carry |
| 14 | Overall posture | **No telemetry, recorded as RISK-0014** with a named trigger | Correct for an internal POC; negligent for a product with users — the difference is written down | Discovering the gap at the moment it starts to matter |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-33 | What the error-boundary fallback screen looks like — copy, which doc 07 atoms it composes, and whether it uses the `base` surface mode or the active theme | Mobile | 1.7 follow-up / the shell scaffold STEP |

Closed by 1.11: **OQ-22** / **OQ-23** / **OQ-26** (wire shapes — doc 11 §5–§7). **OQ-10** is now
expressed concretely as doc 11 §14's Phase-3 checklist plus **OQ-34**; §2.4's correlation-ID
decision is item 2 on that list, and 1.11 deliberately did **not** reserve a header name.

Carried forward, unchanged by this session: **OQ-28** (who owns the sign-off binary),
**OQ-31** / **OQ-32** (Phase 2/3 config and CI). **RISK-0002**'s deferral of this session is
closed by this doc.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.10 | Initial draft from the observability session. Logging is bare `console.*` with a no-bodies rule; metrics, tracing, health checks, correlation IDs, dashboards, alerting, and any vendor are consciously declined with triggers. Adds one shell-root error boundary (doc 03 §4 updated). Opened OQ-33, RISK-0014. |
| v0.1.1 | 2026-08-17 | STEP-1.11 | §2.4's correlation-ID deferral becomes item 2 on doc 11 §14's Phase-3 checklist; no header name reserved. Boundary-logging rule restated in doc 11 §10. Closed OQ-22, OQ-23, OQ-26. |
