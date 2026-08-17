# TypeScript coding standards — quasar-disney-mobile

> **Reconciled to this project in STEP-1.12.** This started as the generic Node/TypeScript default
> that ships with Throughstone; roughly a third of it contradicted decisions already made in
> `architecture/`. The conflicts are resolved below and the reasoning is recorded in
> `architecture/12-test-strategy.md` §10.1. Where a rule here encodes a real decision, the ADR or
> architecture section is linked inline — follow the link before changing the rule.

**Baseline:** TypeScript in **`strict` mode**.

| Runtime | Where | Notes |
|---------|-------|-------|
| **Hermes** | The app, on device (doc 15 §8) | The shipped runtime. No Node APIs — no `node:worker_threads`, no `fs`, no `process.env` at runtime |
| **Node 20+** | Tooling only — Jest, Metro, ESLint, CI (doc 12 §7) | Never assume the app runs here |

Format with **Prettier**, lint with **ESLint** (typescript-eslint) — Prettier owns formatting,
ESLint owns correctness. Both are merge gates (doc 12 §7.1), along with `tsc --noEmit`.

**`noUncheckedIndexedAccess` is off for Phase 1a.** It is genuinely safer, but carousel code is
`resources[0]`-shaped throughout and it would add narrowing friction on every row before a fixed
date. Revisit at Phase 1b. `strict: true` is not optional.

## Naming
- `camelCase` for variables, functions, and methods; `PascalCase` for types, interfaces, classes,
  enums, and React components; `UPPER_SNAKE_CASE` for true constants.
- **Filenames (settled — doc 12 §10.1):** `PascalCase.tsx` for React components, matching doc 07's
  atomic inventory; `camelCase.ts` for everything else — slices, hooks, adapters, factories,
  selectors. Prefer one primary export per file.
- No `I`-prefix on interfaces, no `T`-prefix on types. Booleans read as predicates (`isActive`,
  `hasMore`).
- Wire types match the contract's own casing — `camelCase` fields, per
  `architecture/11-interface-contracts.md` §6.3. Do not "fix" a wire name to match a local
  convention; the contract is the authority.

## Documentation
- **A TSDoc `/** … */` block on every class, function, and method** (the project rule — see
  [`README.md`](README.md)), with `@param`/`@returns` where they add what the signature does not.
  `eslint-plugin-jsdoc` can enforce presence and shape.
- Comment the *why*; let types and names carry the *what*.
- Where a rule in `architecture/` is load-bearing and non-obvious in the code, say so at the site.
  The clearest example: the middleware that reacts to `UNAUTHORIZED` should note that the 401 policy
  deliberately does **not** live in the axios interceptor (**ADR-0017**) — otherwise a developer
  arriving from another RN codebase will look there, not find it, and add a second one.

## Project / module layout
- Source under `src/`, laid out per `architecture/03-architecture-overview.md` §8.1:
  `src/{app,features,shared,api}/`. Import rules in §8.2 are enforced by ESLint
  (`import/no-restricted-paths`), not by review — see doc 12 §7.1.
- **Tests are colocated** (`foo.test.ts` beside `foo.ts`), not in `__tests__/` — settled in doc 12
  §10.1. Test factories are colocated with the module owning the type (doc 12 §4.1) and named
  `*.factory.ts`, imported only from test files so Metro never pulls them into the app bundle.
- **ESM syntax, but no `"type": "module"`** in `package.json` — Metro owns bundling and the flag
  fights the React Native toolchain. **No `dist/`**: `tsc` runs `--noEmit` only.
- Use path aliases over deep `../../..` chains. Keep module internals unexported. Avoid large
  app-wide barrel files.
- `interface` for object shapes, `type` for unions and aliases.

## Types
- **No `any`.** Use `unknown` at boundaries and narrow. Reserve `as` casts for genuinely
  unavoidable cases — never to silence the compiler.
- **This is load-bearing, not style.** `architecture/11-interface-contracts.md` §11.2 and ADR-0016
  make typed fixtures the condition on which the whole type-as-contract strategy rests: mock
  fixtures are declared `Container[]` / `Card[]`, never `any` and never untyped JSON imports. A
  loosely imported `resources.json` makes `tsc --noEmit` — the project's primary contract test —
  verify nothing.
- Let inference work for locals; annotate function signatures and public APIs explicitly.
- Model invalid states out of existence — discriminated unions over optional-field soup.
- Follow the contract's `null`-vs-omitted rule (`api.md`): `null` for a field that always exists but
  may be empty; omitted for one that does not apply to that variant.

## Error handling
- `throw` `Error` (or a subclass), never strings or plain objects.
- **At the API boundary the type is `ApiError { code, status, message }`**
  (`architecture/11-interface-contracts.md` §8.3). Both transports — mock adapter and axios —
  normalize into it, which is what makes the Phase-3 swap invisible above the boundary (ADR-0002).
  Features and hooks **never** see an axios error or a raw rejection.
- Switch on **`code`** (a closed enum), never on `message` and never on a URL path. `message` is
  developer-facing and is never rendered to a user — all user-visible error copy is i18n keyed by
  `code` (doc 11 §7.3, §8.2).
- Preserve causes: `throw new WrappedError("…", { cause: err })`. Don't swallow — handle or rethrow.
- Always handle promise rejections; no floating promises (`@typescript-eslint/no-floating-promises`,
  `no-misused-promises`).

## Logging
**This project uses `console`. There is no logging library, and adding one is a decision to reopen
in `architecture/10-observability.md`, not a style choice.**

Session 1.10 declined the entire telemetry stack — no metrics, tracing, health checks, dashboards,
alerting, or vendor (RISK-0014). Three project decisions make `console` the correct mechanism rather
than a shortcut:

- **DF9:** the analytics hook ships with its final signature backed by `console.log` stubs.
- **Doc 11 §4.4** *requires* a `console.warn` when an unrecognized `Container.variant` arrives — and
  doc 12 §5 spies on it rather than silencing it, so it is asserted behavior.
- **Doc 09 §4.3:** the release build strips `console.*` at compile time, so development logging
  cannot leak into the artifact stakeholders hold.

**The never-log list is binding** (doc 10 §2.3, doc 11 §9.3, doc 04 §5): never log tokens or the
`Authorization` value, never log request or response **bodies**, never log `email`, `password`, or
`userName`. Operation name, `status`, `code`, and duration are fine. Levels follow `console`'s own —
`warn` for recoverable surprises, `error` for failures needing attention.

Revisit only when a real backend and a real crash reporter exist (Phase 3).

## Async
- `async`/`await` gives **concurrency, not parallelism**. There are **no worker threads in React
  Native** — keep CPU-heavy work out of render paths instead.
- Prefer `async`/`await` with `try/catch` over `.then()` chains. Run independent operations
  concurrently with `Promise.all` (fail-fast) or `Promise.allSettled`. The cold-start boot gate is
  exactly this shape: `/me` + HomeFeed + Continue Watching resolve together before first paint
  (doc 04 §6).
- **No floating promises** — every promise is `await`ed, returned, `.catch()`-ed, or `void`-ed.
- All I/O crosses the API module. **No screen or hook imports `axios` or calls `fetch`** — DF1,
  enforced by lint (doc 12 §7.1). Cancellation, when needed, uses axios's own mechanism rather than
  a bare `AbortController` on `fetch`.

## Testing
- **Jest**, React Native preset. `architecture/12-test-strategy.md` is authoritative — tiers in §2,
  the must-cover list in §3, isolation rules in §4, mocking in §5.
- Test files `*.test.ts` / `*.test.tsx`, colocated. One behavior per test, Arrange–Act–Assert.
- Name tests for the behavior (`rejects an expired token`), not the function.
- **Mock only at the native boundary** — `react-native-encrypted-storage`,
  `@react-native-community/netinfo`, `@env`. The mock adapter is a production artifact under test,
  **not** a test double (doc 12 §1, §5). i18n runs for real.
- The store is built per test via `createStore()`; the adapter is constructed per test with latency
  `0`, a frozen clock, and failure injection off by default (doc 12 §4.2, §4.4).
- Keep the suite under **two minutes** (doc 12 §7). Cover behavior and edge cases, not a coverage
  percentage — ~80% is a steer, and there is **no numeric gate** in Phase 1a (doc 12 §8.1).
