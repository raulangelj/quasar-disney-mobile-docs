# Doc 08 — Infrastructure & Deployment

**Version:** v0.1.0
**Status:** Draft
**Coverage:** deferred — server-side hosting, IaC, networking/TLS, and CI are consciously not
enumerated in Phase 1 (see §1). Resurface at each check-in.
**Last updated:** 2026-08-17 (STEP-1.8)
**Audience:** Mobile developers, QA, eng leadership

> Where quasar-disney-mobile runs, how a build reaches the sign-off device, and what actually
> fails on demo day — for a project that deliberately ships no server.

## Table of Contents

1. [Scope: there is nothing to host](#1-scope-there-is-nothing-to-host)
2. [Hosting & compute — the build machine is the infrastructure](#2-hosting--compute--the-build-machine-is-the-infrastructure)
3. [Deploy pipeline — build → install → roll back](#3-deploy-pipeline--build--install--roll-back)
4. [Infrastructure as code — lockfiles only](#4-infrastructure-as-code--lockfiles-only)
5. [Networking, TLS, secrets](#5-networking-tls-secrets)
6. [Resilience — what actually fails on demo day](#6-resilience--what-actually-fails-on-demo-day)
7. [Backups & DR — artifact and code survival](#7-backups--dr--artifact-and-code-survival)
8. [Cost & cloud coupling](#8-cost--cloud-coupling)

---

## 1. Scope: there is nothing to host

Docs 03 and 05 are unambiguous: **we build no backend**, and a future HTTP API is not a
component of this system — it is a Phase-3 swap behind the API module. All Phase-1 I/O is
in-process mocks. There is therefore **no server, no database, no queue, no cache, and no
runtime infrastructure to provision, secure, scale, or pay for.**

This doc is scoped accordingly to the **mobile build-and-distribute path**: what machine
produces the binary, how it reaches a device, and how you undo it.

**Explicitly out of scope, by decision:**

| Not covered | Why | Who owns it |
|-------------|-----|-------------|
| Application hosting (cloud provider, region, PaaS/K8s/serverless) | Nothing to run | — (does not exist) |
| Phase-3 backend hosting, provider choice, residency | We integrate against it; we do not build or operate it | Backend team (Phase 3) |
| Terraform / Pulumi / provider tooling | No cloud resources to declare | — |
| Load balancers, DNS, ingress, TLS certificates | The app reaches no host | — |
| Production secret manager | There is no production tier | — |
| Uptime SLO, on-call, paging | No service to keep up | — |

We deliberately do **not** pre-decide a Phase-3 hosting posture as guidance for the backend
team. Recommending a provider for infrastructure we neither build nor operate would be
architecture theatre, and it would age badly before Phase 3 starts.

**Revisit trigger:** Phase 3 backend integration — at which point the *client-side* half
(base URL configuration, HTTPS/ATS posture, pinning per RISK-0007) reopens here, and the
server half belongs to the backend team's own architecture work.

## 2. Hosting & compute — the build machine is the infrastructure

With no server, "compute" is the machine that produces the artifact.

| Phase | Build host | Artifact destination |
|-------|-----------|----------------------|
| **Phase 1a (now)** | Each developer's laptop — Xcode + Android Studio | Simulator/emulator for development; **USB-installed release build** on the sign-off device |
| **Phase 2** | Bitrise hosted runners (macOS required for iOS signing) | Installable QA builds — externally blocked on accounts and signing (doc 02 §5, OQ-05) |
| **Phase 3+** | Unchanged for the client | Store/TestFlight only if the product leaves internal use (**RISK-0005** must be resolved first) |

### The sign-off artifact is a release build

Docs 15 §8 and 05 §2 both treat a **release build** as the *fallback* if a device hitches at
sign-off. **This session inverts that: the release build is the declared sign-off artifact,
not the contingency.**

| | Debug build | **Release build (sign-off)** |
|---|---|---|
| JS bundle | Served live by Metro | **Embedded in the binary** |
| Live dependency during the demo | Metro dev server + the laptop hosting it | **None** |
| Failure mode on stage | Red box, dev-server disconnect, slow first paint | Same as any shipped app |
| Role | Development loop | **The thing stakeholders see** |

Concretely: iOS `Release` scheme; Android `assembleRelease` / `bundleRelease`, debug-signed
(no distribution certificate is needed for a USB install); JS bundle embedded; **no Metro
attached**; installed on the demo device before the session begins.

**Rationale.** The demo's subject is craft and visual fidelity. A debug bundle is slow, keeps
the developer's laptop in the demo's critical path, and can red-box. A hitch is a worse
outcome than nobody notices; a red box is the demo failing. The cost is ~30 minutes of
first-time setup (Android release signing config, one iOS release run), and it belongs at the
**Tuesday-AM sync point** (doc 02 §8), not on the morning of the 18th.

**Forecloses:** demoing from a simulator with Metro attached; treating "use a release build"
as an in-the-moment remedy rather than a prepared step.

## 3. Deploy pipeline — build → install → roll back

There is no pipeline in Phase 1a. There is a person and a cable. The method's rule applies:
**manual is fine if it is written down and repeatable** — so it is written down, in
`runbooks/release-deploy.md`, rewritten by this session from the generic template into the
actual Phase-1a procedure.

```
git tag  →  version stamp  →  clean  →  release build (iOS + Android)
        →  install on demo device(s) via USB  →  smoke both flows
        →  keep the previous known-good binary on disk
```

### Version stamping is the deploy identity

Doc 15 §7 already requires stamping `CFBundleShortVersionString` / `versionName`. This session
makes it a **step in the runbook**, paired with a **git tag** on the commit that produced the
build — so "which build is on the demo phone, and what commit is it?" has an answer. Without
it, a demo-day bug report cannot be tied to source.

### Rollback is reinstall, not rebuild

**Keep the last known-good `.ipa` / `.apk` on disk** and reinstall it. A rebuild-to-roll-back
takes ten minutes or more and can fail for a *new* reason under exactly the pressure you can
least afford it; reinstalling a file that already ran takes about a minute. This is the
cheapest insurance available for the 18th.

**Rollback trigger:** any smoke-test failure on the demo device — the flows in doc 02 §4
criteria F1–F3. Decide fast; investigate afterwards.

### Phase 2 (Bitrise) — deliberately not designed here

CI design is left to the Phase-2 STEP that actually implements it, including the `npm audit`
gate that doc 06 §6 parked in this session (see §5). `bitrise.yml` will then be the project's
first genuine pipeline-as-code artifact, committed like any other source.

## 4. Infrastructure as code — lockfiles only

No cloud resources exist to declare, so Terraform/Pulumi and equivalents are N/A. The
*problem* IaC solves — "it works on my machine and nobody can reproduce it" — is nonetheless
live, and is precisely **RISK-0003**: two seniors on two laptops with drifting Xcode, Node,
JDK, Ruby, and CocoaPods versions.

**Decision: the simplest option that costs nothing — commit the lockfiles the React Native
template already generates.** No authored pin files, no toolchain table, no tooling.

| Committed (already produced by the scaffold) | Pins |
|---|---|
| `package-lock.json` | JS dependencies — already mandated by doc 06 §6 |
| `Podfile.lock` | iOS native pods |
| `Gemfile.lock` | CocoaPods itself (RN's own convention) |
| `gradle/wrapper/*` | Gradle version |

The rule is simply **do not gitignore them**.

**Consciously not done**, given this is a showcase with no QA or deployment tier: `.nvmrc`,
`.ruby-version` authored by hand, and a README toolchain table naming known-good Xcode / JDK
versions.

**Accepted residual:** **Xcode version drift between the two laptops is unmitigated.** Xcode
cannot be pinned by a committed file — only stated and agreed — and we chose not to state it.
If the two machines disagree, it surfaces during the scaffold window. This sits inside
**RISK-0003**'s existing blast radius and mitigation (pair on the scaffold; drop Android for
Saturday if it slips) rather than opening a new risk.

## 5. Networking, TLS, secrets

| Area | Phase 1 | Later |
|------|---------|-------|
| Load balancer / ingress / DNS | **N/A** — no host to reach | Backend team, Phase 3 |
| TLS certificates | **N/A** | Backend team, Phase 3 |
| iOS ATS | **Left at its default**; no cleartext exception added | Nothing to undo when a real HTTPS host arrives |
| Certificate pinning | **None** — **RISK-0007** | API-module interceptor, Phase 3 |
| Public vs. private surfaces | Neither — the app exposes no listening surface | — |

### Secrets

Doc 06 §5 already settled this and this session does not reopen it: **no secret manager,
because there is no production tier.**

- Gitignored `.env` holds `API_BASE_URL`, `DEMO_EMAIL`, `DEMO_PASSWORD`.
- The repo commits `.env.example` with placeholders only (`templates/env-example.txt`).
- The mock adapter reads the demo pair from env — not a committed fixture, not a screen-level `if`.

**Operational consequence this session adds:** the release build reads the demo credentials
from `.env`, so **`.env` must exist on whichever laptop builds the sign-off binary.** A missing
`.env` produces a login that fails for the wrong reason, on stage, looking exactly like a bug in
the auth flow. This becomes a **pre-flight checkbox** in `runbooks/release-deploy.md` — a
checklist item, not a new mechanism.

### Dependency audit in CI (OQ-27 / RISK-0010)

Doc 06 §6 deferred the exact `npm audit` gate — fail vs. warn, lockfile check, cadence — to
this session. **This session consciously does not decide it.** A fail-vs-warn threshold is
meaningless without a CI system to enforce it, and Bitrise is Phase 2 and externally blocked.

**OQ-27 is closed as "deliberately not decided in 1.8"**, and RISK-0010's revisit trigger moves
to *Bitrise implementation* rather than *session 1.8*. What stands in the meantime is unchanged
and adequate for the blast radius: committed lockfile, vet-before-add
(`runbooks/dependency-supply-chain.md`), and `npm audit` on the check-in cadence.

## 6. Resilience — what actually fails on demo day

The availability question for this project is not "what is our uptime target" — there is no
service and no uptime concept. It is **"what stops the 18 Aug sign-off from happening."**

### Single points of failure

| SPOF | Consequence | Mitigation |
|------|-------------|------------|
| **The demo device** | No demo | **Install the release build on two devices** (or a device plus a second device/simulator) |
| **The demo laptop** (holds `.env` and the signed build) | Cannot rebuild or reinstall | A second machine can build; `.env` present on both |
| **The `.env` file** | Login fails on stage for a non-code reason | Pre-flight checkbox (§5) |
| **The git remote** | Lost work, no rollback source | Push before the demo (§7) |
| ~~Metro dev server~~ | ~~Red box / disconnect mid-demo~~ | **Eliminated** by the release-build decision (§2) |
| **Network availability at the venue** | See below — the sharp one | §6.1 |

**Availability target: none.** There is no service to keep up. Recovery is measured in
reinstalls, not in nines.

### 6.1 The connectivity gate is a demo-day failure mode

Doc 15 §2 makes the no-internet overlay **shell-owned and unconditional**: when NetInfo reports
no usable network, a full-screen overlay covers whichever navigator is mounted.

But **all Phase-1 data is in-process mocks — the app needs no network to function.** So a demo
room with no wifi, or a captive-portal guest network, yields a fully working application
displaying a blocking *"Es necesario revisar tu conexión a internet"* screen at sign-off. The
overlay would be **correct by specification and fatal by outcome**.

This gives **OQ-21** (doc 15: does "usable network" mean *interface up* or *internet
reachable*?) real teeth. It had been parked as a foundation-STEP implementation detail; it is
in fact a demo-day decision.

**Resolved (this session): "usable network" means the interface is up.** Plain NetInfo
`isConnected` — no internet-reachability probe, no captive-portal detection. See **ADR-0014**.

- **Cheaper to implement** — no probe, no timeout tuning, no false-negative handling.
- **A captive-portal venue wifi does not trip the overlay**, which is the realistic 18 Aug
  hazard.
- **Accepted tradeoff:** a device connected to a portal it has not authenticated through
  reports "connected" and the overlay stays hidden. In Phase 1 that is harmless — there is no
  real request to fail. It becomes a genuine (and reopenable) question in Phase 3, when a real
  host means "connected but not reachable" produces confusing fetch failures instead of a clear
  gate.
- **Belt and braces:** "demo device has a live network connection" is a pre-flight checkbox in
  the release runbook regardless.

### Graceful degradation

Already covered by existing decisions and not re-litigated here: request failure is a feature
concern (inline auth error; storefront error/empty state), no-network is a shell concern
(overlay + auto-restore + `REINTENTAR`), and a corrupt persist blob purges rather than crashes
(doc 04). Doc 15 §2's rule stands — **do not conflate "no network" with "request failed."**

## 7. Backups & DR — artifact and code survival

Reframed for a project whose only durable assets are *the source* and *the binary*. There is no
database, so there is nothing to snapshot and no schema migration to reverse.

| | What | Target |
|---|------|--------|
| **What is backed up** | Source (git remote) and the last known-good `.ipa` / `.apk` on disk | — |
| **Cadence** | Push before the demo; keep each known-good binary as it is produced | — |
| **Retention** | The previous known-good binary, at minimum — one is enough to roll back to | — |
| **RPO** (tolerable loss) | Last git push | Push before the demo |
| **RTO** (time to recover) | Reinstall the known-good binary over USB | **~1 minute** |
| **Restore rehearsal** | **Self-proving** — the binary you are holding already installed and ran once. That *is* the rehearsal | No separate drill |

**Who does what when it is down:** whoever holds the demo device reinstalls the previous
binary. There is no escalation path because there is no one to escalate to and nothing else to
try. If the previous binary also fails, the demo runs from the second device (§6).

This is a deliberately minimal DR posture, correct for an internal POC. It would be negligent
for a product with users, and the trigger for revisiting it is exactly that: **real users, real
data, or a store release.**

## 8. Cost & cloud coupling

**Cost: $0.** No cloud spend, no managed services, no per-seat tooling. Cost drivers as this
scales are entirely Phase-3-and-later concerns owned by others (backend hosting, artwork CDN).
Phase 2 introduces the first real line item — Bitrise build minutes — which is why OQ-06
(budget) remains open and unblocking.

**Coupling:**

| Coupled to | Depth | Painful to move? |
|-----------|-------|------------------|
| Apple toolchain (Xcode, signing) | Inherent to shipping an iOS app | Not avoidable, not worth avoiding |
| Google toolchain (Android SDK, Gradle) | Inherent to shipping an Android app | Same |
| Bitrise (Phase 2) | **Shallow** — any macOS CI runs the same commands | Low: the build is `xcodebuild` + `gradlew`, not Bitrise-specific logic |
| Any cloud provider | **None** | N/A |

**There is no cloud lock-in to avoid, because there is no cloud.** The one coupling decision
that Phase 2 will make — Bitrise — stays shallow as long as the CI config *invokes* the
project's build commands rather than encoding build logic in vendor steps. That is the single
guardrail worth carrying into Phase 2.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Scope of this doc | **Mobile build-and-distribute only.** No application hosting exists; Phase-3 backend hosting is explicitly the backend team's | Docs 03/05 build no server — there is nothing to host, secure, or scale | Pre-deciding a Phase-3 provider/region as "guidance"; inventing infrastructure we do not own |
| 2 | Build host | Dev laptops now; Bitrise hosted runners in Phase 2 (externally blocked) | iOS signing needs macOS; no CI exists yet | A Phase-1a CI pipeline |
| 3 | Sign-off artifact | **Release build on-device is the declared demo artifact**, not the fallback | Removes Metro and the laptop from the demo's critical path; a red box is worse than any hitch | Demoing from a simulator with Metro attached; ~30 min setup at the Tue-AM sync point |
| 4 | Deploy procedure | Manual, **written down** in `runbooks/release-deploy.md` (rewritten from the generic template) | Method rule: manual is fine if repeatable. Makes "Dev A is unavailable Tuesday" survivable | An automated pipeline before Bitrise exists |
| 5 | Deploy identity | Version stamp **paired with a git tag** per build | Ties the binary on the demo phone to a commit | Untraceable builds |
| 6 | Rollback | **Reinstall the previous known-good `.ipa`/`.apk`** — do not rebuild | ~1 min vs. 10+ min, and a rebuild can fail for a new reason under pressure | Keeping only the newest artifact |
| 7 | Infrastructure as code | **Lockfiles only** — commit what the RN scaffold already generates. No `.nvmrc`, no toolchain table, no IaC tooling | Showcase with no QA/deployment tier; zero added cost | Xcode drift stays unmitigated (folded into RISK-0003) |
| 8 | Networking & TLS | **N/A.** ATS left at default; no cleartext exception | No host to reach; nothing to undo in Phase 3 | — |
| 9 | Production secrets | **No secret manager.** Gitignored `.env` + committed `.env.example` (doc 06 §5 unchanged) | No production tier exists | Adds a pre-flight check: `.env` must exist on the build laptop |
| 10 | `npm audit` CI gate | **Consciously not decided in 1.8.** OQ-27 closed as deferred; RISK-0010 revisit moves to Bitrise implementation | A fail/warn threshold is meaningless without CI to enforce it | Leaving OQ-27 dangling against a session that cannot answer it |
| 11 | SPOFs | The **demo device, demo laptop, and `.env`** — not infrastructure. Mitigate by duplication: two devices, two build-capable machines | Matches the real blast radius | Treating this as an availability-engineering problem |
| 12 | Availability target | **None** — no service, no uptime concept | Nothing runs continuously | — |
| 13 | Connectivity gate semantics (**OQ-21**) | **"Usable network" = interface up** (NetInfo `isConnected`); no reachability probe | Cheaper, and a captive-portal venue wifi cannot block a demo whose data is all in-process mocks (**ADR-0014**) | Connected-but-not-reachable reports online — harmless in Phase 1, reopens in Phase 3 |
| 14 | Backups & DR | Git remote (source) + previous known-good binary (artifact). **RPO** = last push, **RTO** ≈ 1 min. Restore rehearsal is self-proving | No database, no migrations; the only assets are source and binary | Would be negligent for a product with real users — revisit at real data or a store release |
| 15 | Cost & coupling | **$0**; no cloud coupling. Bitrise (Phase 2) stays shallow if CI **invokes** build commands rather than encoding build logic | Nothing to optimize or escape | Vendor-specific build logic in `bitrise.yml` later |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-21~~ | ~~NetInfo "usable network": interface up vs. internet reachability (captive portal)~~ **Resolved (1.8):** interface up is enough — ADR-0014 | — | closed |
| ~~OQ-27~~ | ~~Exact dependency-audit gates once Bitrise exists~~ **Closed (1.8) as deliberately deferred:** undecidable without CI; RISK-0010 revisit trigger moves to Bitrise implementation | — | closed |
| OQ-28 | Who owns building and installing the sign-off binary, and on which two devices? | Eng leadership | Tue-AM sync point (doc 02 §8) |
| OQ-29 | Does the Phase-3 "connected but not reachable" case need a reachability probe once a real host exists? | Mobile | Phase 3 backend integration |

Carried forward: OQ-05 (Bitrise setup → Phase 2), OQ-06 (budget — Bitrise minutes are the first
real line item), OQ-13 (wordmark outlining before 18 Aug).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.8 | Initial draft from the infrastructure & deployment session. Release build declared the sign-off artifact; release/rollback procedure written to `runbooks/release-deploy.md`; OQ-21 resolved (ADR-0014); OQ-27 closed as deferred. Opened OQ-28, OQ-29. |
