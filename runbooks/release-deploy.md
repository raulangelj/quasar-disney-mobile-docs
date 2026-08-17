# Runbook — Release, Deploy & Rollback

> **How to run:** This is an **operational procedure, not a STEP**. Run it whenever you produce
> a build that someone other than you will see — tell your agent *"run the release"* (or
> *"roll back the last release"*) and it follows this file.
>
> **Project-specific.** This file was rewritten from the method's generic template in
> **STEP-1.8** to match what quasar-disney-mobile actually does. The mechanism it executes is
> designed in `architecture/08-infrastructure-deployment.md` (§2 sign-off artifact, §3 deploy
> pipeline, §6 SPOFs). Environments are session 1.9 — until then there is exactly one
> destination: **a physical demo device over USB**.

## What a "release" means here

There is **no server, no environment tier, and no store** (doc 08 §1, doc 15 §7). A release is:

> a **release-configuration build**, installed over USB onto the device that stakeholders will
> hold.

The primary use is the **2026-08-18 stakeholder sign-off** (doc 02 §4). Do this at the
**Tuesday-AM sync point**, not on the morning of the demo — first-time release signing setup
takes about 30 minutes and you do not want to discover that with an audience waiting.

## Why this runbook exists

The failure mode is not the build. It is having **no planned way back** when the thing on the
device misbehaves in front of the people whose sign-off the project exists to get. Decide the
undo before you build, keep the artifact you are replacing, and the worst case is a
sixty-second reinstall instead of improvised debugging on stage.

---

## Part 1 — Pre-flight

- [ ] **Work is merged and the test suite is green** — every reducer, middleware, hook, and the
      API/mock layer (doc 02 criterion A6), not just the area you touched.
- [ ] **`.env` exists on this machine**, with `API_BASE_URL`, `DEMO_EMAIL`, `DEMO_PASSWORD`
      (doc 06 §5). **Check this explicitly.** The mock adapter reads the demo pair from env, so
      a missing `.env` produces a login that fails on stage looking exactly like an auth bug.
      `.env` is gitignored — a fresh clone does **not** have it.
- [ ] **Version stamped** — bump `CFBundleShortVersionString` (iOS) and `versionName` (Android),
      per doc 15 §7.
- [ ] **Git tag the commit** you are about to build, so the binary on the device is traceable to
      source. Push the tag.
- [ ] **Push to the remote.** This is the project's entire backup posture — RPO is "last push"
      (doc 08 §7).
- [ ] **The previous known-good `.ipa` / `.apk` is on disk and you know where.** This is the
      rollback (Part 4). If there isn't one yet, this is the build that becomes it.
- [ ] **Rollback plan confirmed:** reinstall the previous binary. If you cannot point at that
      file right now, you are not ready to build.

*No migration checks — there is no database. No secrets-manager step — there is no production
tier (doc 08 §5).*

## Part 2 — Build

Build **release configuration** on both platforms. Debug builds are the development loop and are
**not** what gets demoed (doc 08 §2): the release build embeds the JS bundle, so Metro and the
laptop hosting it are out of the demo's critical path.

- [ ] **Clean** first — stale native artifacts are the classic source of a build that works only
      on the machine that has been building all week.
- [ ] **iOS:** `Release` scheme. Debug/local signing is fine — this is a USB install, not a
      distribution build. No distribution certificate or provisioning profile is required.
- [ ] **Android:** `assembleRelease` (or `bundleRelease`), debug-signed. First time on a fresh
      checkout, this needs release signing configured in Gradle — budget ~30 minutes.
- [ ] **Confirm the JS bundle is embedded** and **Metro is not running / not attached**. If the
      app only works while Metro is up, you built debug.
- [ ] **Keep both artifacts.** Copy the `.ipa` and `.apk` somewhere durable alongside the tag
      name. These become the next release's rollback target.

## Part 3 — Install & verify

- [ ] **Install over USB onto the demo device.**
- [ ] **Install onto a second device too.** The demo device is a single point of failure
      (doc 08 §6); a second installed device is the cheapest possible redundancy.
- [ ] **Confirm the demo device has a live network connection.** The shell's connectivity
      overlay keys on interface state (ADR-0014), so an associated wifi or cellular interface is
      enough — a captive portal will not trip it. Check anyway; the overlay is full-screen and
      blocking if it does appear.
- [ ] **Smoke the launch criteria** (doc 02 §4) on the demo device, on **both platforms**:
      - **F1** — welcome → credentials → home completes end to end.
      - **F2** — wrong credentials produce the **inline error state** (red underline + message),
        not an alert, not a crash.
      - **F3** — home renders both carousel variants from mock data; tapping a card alerts with
        the title.
- [ ] **A11y spot-check** — the ~10-minute VoiceOver + TalkBack pass (doc 07 §8, RISK-0011).
      The live-region auth error is the one that fails silently.
- [ ] **Record what shipped:** tag, platform, device, date, who built it.

*No production signals to watch, no health checks, no error-rate window — there is no service
(doc 08 §1). Verification is the smoke test on the device in your hand.*

## Part 4 — Rollback

**Trigger.** Any smoke-test failure in Part 3, or any defect found on the demo device that you
cannot explain in a couple of minutes. Decide fast — roll back and investigate afterwards.

**Mechanism — reinstall, do not rebuild.**

1. Install the **previous known-good `.ipa` / `.apk`** from disk over USB.
2. Re-run the Part 3 smoke checks on it.
3. Note which tag you reverted to.

A rebuild-to-roll-back takes ten minutes or more and can fail for a **new** reason under exactly
the pressure you can least afford it. Reinstalling a file that already ran takes about a minute
(doc 08 §3).

**If the previous binary also fails:** run the demo from the **second device** (Part 3) and stop
touching the primary until after the session.

**After a rollback.** Something broke that the test suite did not catch. Capture symptom,
trigger, and what you did, then hand off to `runbooks/incident-postmortem.md` so the real fix
does not evaporate once the demo is over.

## After the release

- [ ] Confirm the shipped version is **tagged and recorded**.
- [ ] **Phase milestone?** If this build is the 1a sign-off, the method asks about **release
      notes** and user-facing docs (`METHOD.md` §5). For an internal POC the honest answer is
      usually "no" — but answer it deliberately rather than skipping it.
- [ ] Note anything for the next **check-in** (`runbooks/check-in.md`) — especially build steps
      that were more painful than this runbook implies, so the next person gets the corrected
      version.

---

## When this runbook changes

**Phase 2 (Bitrise)** replaces Parts 2 and 3 with a pipeline and installable QA builds — rewrite
those parts then, and design the CI gates (including the deferred `npm audit` gate, RISK-0010)
in that STEP. **Session 1.9** introduces environments, which is when "which destination" becomes
a real question. Until either happens, this file is the whole deployment story.
