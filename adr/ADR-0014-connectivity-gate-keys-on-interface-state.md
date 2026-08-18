# ADR-0014: The connectivity gate keys on interface state, not internet reachability

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- architecture/08-infrastructure-deployment.md §6.1
- architecture/15-native-app-architecture.md §2 (connectivity gate; OQ-21)
- architecture/02-phasing-roadmap.md DF11
- ADR-0004 — Online-only client with a shell-level connectivity gate

## Context

ADR-0004 and doc 15 §2 give the app shell an unconditional, full-screen no-internet overlay:
when `@react-native-community/netinfo` reports no usable network, the overlay covers whichever
navigator is mounted, and it hides again on reconnect or `REINTENTAR`.

Doc 15 left **OQ-21** open: what counts as a "usable network"? Two readings were available.

1. **Interface up** — NetInfo's `isConnected`: a wifi or cellular interface is associated.
2. **Internet reachable** — additionally probe that traffic actually reaches the internet, so a
   captive portal (hotel/venue/conference wifi that intercepts requests until you authenticate)
   counts as *offline*.

OQ-21 had been parked as a foundation-STEP implementation detail. Session 1.8 found it is
actually a **demo-day availability decision**, because of a property specific to Phase 1:

> **All Phase-1 data is in-process mocks. The app needs no network whatsoever to function.**

That inverts the usual tradeoff. Normally the reachability probe is the more *correct* choice —
it prevents an app from confidently issuing requests that will hang against a captive portal.
Here there are no requests to protect: the overlay guards nothing, and every minute it is up is
a minute of a working application refusing to show itself.

The concrete hazard is the 2026-08-18 stakeholder sign-off (doc 02 §4, P1). A demo room with a
captive-portal guest network — or no wifi at all — would produce a fully functional app
displaying a blocking *"Es necesario revisar tu conexión a internet"* screen. The overlay would
be **correct by specification and fatal by outcome**, on the one occasion the project exists to
survive.

## Decision

**"Usable network" means the interface is up.** The shell's connectivity gate keys on plain
NetInfo `isConnected`. No internet-reachability probe, no captive-portal detection, no
HTTP HEAD to a known endpoint, no timeout tuning.

**Rationale.** It is both the cheaper implementation and the safer one *for this phase*. There
is no request the overlay protects, so its only achievable effect in Phase 1 is a false
positive — and the highest-consequence false positive lands on demo day.

**Alternatives.**
- *Reachability probe.* Rejected for Phase 1: it adds a probe endpoint, a timeout policy, and
  false-negative handling, in exchange for protecting network calls that do not exist. It also
  creates the exact captive-portal block described above.
- *Remove the overlay for Phase 1.* Rejected — DF11 and ADR-0004 make the gate a
  visual-fidelity requirement matching `inputs/ui/07-no-internet.png`, and the reference screen
  is part of what stakeholders sign off on.
- *Debug flag to bypass the overlay.* Rejected — a demo-only escape hatch is a code path nobody
  tests, and it would sit in the shell where it could ship.

**Reversibility: low cost.** The change is one predicate inside the shell's connectivity
selector. No screen, feature, or contract depends on how the predicate is computed.

## Consequences

**Easier.** The gate is a few lines of shell code with no network dependency of its own. A
captive-portal or no-wifi venue cannot block the 18 Aug sign-off. Removing the reachability
probe removes the only Phase-1 component that would have needed a network endpoint to talk to —
in a project that otherwise has none.

**Harder / accepted.** A device associated to a network it cannot actually reach through reports
"connected," so the overlay stays hidden. In Phase 1 this is harmless: no real request exists to
fail. In **Phase 3**, when the API module talks to a real host, "connected but not reachable"
stops being harmless — the user would get opaque fetch failures on the credentials screen and
the storefront instead of the clear, designed offline gate. That case is recorded as **OQ-29**
against Phase 3 backend integration, and is the trigger to revisit this ADR.

**New work.** None in Phase 1. Independently of this decision, "demo device has a live network
connection" is a pre-flight checkbox in `runbooks/release-deploy.md`, so the gate is not relied
on as the only line of defense.

**Unchanged.** ADR-0004 stands in full: online-only, shell-owned overlay, auto-restore by auth
state, `REINTENTAR` as the manual path, and doc 15 §2's rule that **"no network" is a shell
concern and "request failed" is a feature concern** — this ADR narrows only how the shell
answers the first question.
