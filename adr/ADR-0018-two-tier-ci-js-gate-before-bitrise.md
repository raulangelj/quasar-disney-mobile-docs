# ADR-0018: A JavaScript CI gate from day one; Bitrise remains the native-build tier

**Status:** Accepted
**Date:** 2026-08-17

## Related documents

- `architecture/12-test-strategy.md` §7 (CI gates), §4.3 (`@env` in tests — the enabling decision)
- `architecture/11-interface-contracts.md` §11.4 (the "none until Bitrise" line this ADR amends)
- `architecture/09-environments.md` §2 (CI is a runner, not an environment), §7.1 (trunk-only release builds)
- `architecture/08-infrastructure-deployment.md` §2 (Bitrise is Phase 2, externally blocked)
- `architecture/02-phasing-roadmap.md` §9 (the two-developer split and its two sync points)
- OQ-05 (Bitrise setup), OQ-18 (application repo name), RISK-0010 (`npm audit` gate)

## Context

Doc 11 §11.4 recorded the Phase-1 CI posture in one line: **"None until Bitrise"** (Phase 2,
**OQ-05**), with pre-merge checks run locally as `tsc --noEmit` plus the test suite. Doc 08 §2 places
Bitrise in Phase 2 because it is **externally blocked** — accounts, certificates, provisioning
profiles, signing.

Session 1.12 found that this conflates two things that have different blockers.

Bitrise is blocked because it builds **native binaries**. But the test tiers this project actually
ships in Phase 1a — `tsc --noEmit`, Jest unit tests, Jest integration tests over a real store and an
in-process mock adapter (doc 12 §2) — are **plain Node**. They need no simulator, no Xcode, no
certificate, no device. They run on `ubuntu-latest`.

They also, as of doc 12 §4.3, need **no secrets**. Because tests resolve `@env` to a committed stub
rather than the developer's gitignored `.env`, there is nothing for a runner to be missing. Had that
decision gone the other way, hosted CI would have been impossible in Phase 1 regardless of Bitrise —
the two decisions are coupled, and this one depends on that one.

The schedule sharpens the value. Doc 02 §9 puts two developers on disjoint branches with exactly
**two hard sync points**, the first being Saturday end of day when the foundation and the contract
merge, and promises "no coordination" on Sunday and Monday. That promise rests on the branches
genuinely being independent. A merge gate is what detects the moment they stop being independent — at
the merge, rather than at the Tuesday-morning integration where doc 02 §8 has no recovery time
before an immovable sign-off.

The counter-argument, weighed: `templates/ci/README.md` warns that an unconfigured gate which
silently passes is worse than none, and any gate spends setup time that Phase 1a's ~3.5 days do not
obviously have. Against that, the workflow is a stamped template plus four commands.

## Decision

**1. CI is two tiers, distinguished by what they need rather than by phase.**

| Tier | Runner | What | Phase |
|------|--------|------|-------|
| **A — JS gate** | GitHub Actions, `ubuntu-latest`, every push + pull request | `tsc --noEmit` · `jest` · `eslint` · `prettier --check` | **1a**, from the scaffold STEP |
| **B — Native build** | Bitrise | Native build, installable QA artifacts, release smoke | **Phase 2**, externally blocked (OQ-05) |

**2. Tier A is stamped from `templates/ci/code-repo-ci.yml`** into the application repo's
`.github/workflows/ci.yml` when that repo is scaffolded (OQ-18), replacing the template's deliberate
`exit 1` step. The application repo is hosted on GitHub, confirmed in this session.

**3. Tier A blocks merge, and it enforces four architecture criteria that were previously
review-only:** DF5 / criterion A5 (no cross-feature imports) and DF1 (no `axios`/`fetch` outside
`src/api/`) via `import/no-restricted-paths`; criterion A2 (no hardcoded design values) via a
pattern-matching rule whose reach is bounded and recorded as **RISK-0017**; and criterion A6 via the
suite itself. The full gate list is doc 12 §7.1.

**4. Doc 11 §11.4's "none until Bitrise" is amended** to this split. Its substantive claims survive:
the commands are still `tsc --noEmit` plus the suite, and there is still **no OpenAPI/schema
linting**, because no such artifact exists by decision (ADR-0016).

**5. This adds no environment.** Doc 09 decision 3 holds — CI is a *runner* of checks, with no
distinct config, data, or audience. Tier A gets a promotion-flow row (doc 09 §7.1), not an
environments row.

**Rationale.** The cost is one stamped workflow file and four commands; the benefit is that the
riskiest merge in the schedule is verified by a machine rather than by two tired people at the end of
a Saturday. Deferring it to Phase 2 would defer it past the only dates on which it matters.

**Alternatives.** *Keep "none until Bitrise"* was rejected because it defers a gate past the deadline
it would protect, on the strength of a blocker (signing) that the JS suite does not encounter. *One
tier, all of it at Bitrise* was rejected for the same reason and additionally because it would make
every future test run depend on an external account that does not yet exist. *A local pre-commit hook
instead* was considered: it is cheaper still, but it is bypassable, unenforced on a teammate's
machine, and invisible at the merge — which is exactly the moment this exists to cover.

**Reversibility.** High. Deleting a workflow file is trivial, and **OQ-35** already asks whether
Tier B subsumes Tier A when Bitrise lands.

## Consequences

**Easier.** A red trunk is visible before Tuesday. Four architecture criteria stop depending on
review discipline during the weekend when review discipline is thinnest. Coverage is reported on
every pull request (doc 12 §8.1) without a coverage vendor. And the Phase-2 Bitrise STEP inherits a
working, already-trusted test command instead of authoring one under its own pressure.

**Harder.** A merge gate blocks *both* developers when trunk is red, which sits in mild tension with
doc 02 §9's "no coordination needed on Sunday and Monday." That tension is real and is recorded as
**OQ-36** — who owns a red trunk during the parallel window — rather than papered over. It is also
the argument for doc 12 §7's under-two-minute speed budget: a gate that is both blocking and slow is
one people route around.

**New obligation.** The scaffold STEP must stamp and configure the workflow and the ESLint rules
(doc 12 §11, items 3 and 4). An unconfigured `code-repo-ci.yml` fails by design, so the failure mode
is loud rather than silent — which is the behavior the template intends.

**Unchanged.** `npm audit` stays out of the gate (**RISK-0010**, decided in the Bitrise STEP); no
coverage threshold is introduced (doc 12 §8.1, **RISK-0016**); and the release build's gate is still
doc 09 §7.1's tagged trunk commit plus the runbook's pre-flight and smoke.
