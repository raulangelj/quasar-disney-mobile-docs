# Doc 05 — Scaling & Performance

**Version:** v0.1.0
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.5)
**Audience:** Mobile developers, backend team, QA

> Load this React Native demo actually has, which on-device shortcuts are fine, and which
> cheap boundaries keep a later production catalog from forcing a rewrite.

## Table of Contents

1. [Load profile](#1-load-profile)
2. [Performance targets](#2-performance-targets)
3. [Stateful vs. stateless](#3-stateful-vs-stateless)
4. [Bottlenecks & scaling strategy](#4-bottlenecks--scaling-strategy)
5. [Caching & async](#5-caching--async)
6. [Don't-foreclose](#6-dont-foreclose)
7. [Storefront payload (scale-relevant shape)](#7-storefront-payload-scale-relevant-shape)

---

## 1. Load profile

There is **no backend to size** in this repo. Load is devices running in-process mocks.

| Horizon | Users / sessions | Data / requests |
|---------|------------------|-----------------|
| **Launch (Phase 1a, 2026-08-18)** | **&lt;10 concurrent sessions** — two seniors + stakeholders, typically one device or simulator at a time | Mock catalog: HomeFeed first page ≈ **16 containers** (one `hero` + 15 others) + a separate Continue Watching `Container[]` (typically one `progress` container). Tiles page. JWT-gated `/me` + two feed GETs on boot |
| **~12 months / 100× “if it works”** | Still internal as *this* product. The 100× case is the **production streaming app this templates**: **thousands of concurrent clients**, real catalog and artwork CDN | Same client boundaries; **HTTP** behind the API module. We still **build no backend** |

Phase 1 numbers do not change the stack. They only decide which shortcuts are cheap vs. which would force a rewrite at production load.

---

## 2. Performance targets

Doc 01 dropped performance from v1 success criteria. Doc 15: **no numeric SLOs** (FPS, binary
size, cold start, battery) as an 18 Aug gate. If a device hitches at sign-off, use a **release
build** — do not open a performance program.

Targets below are **perceived UX** for the operations someone waits on. Skip search, ingest,
p99, throughput, and battery.

| Operation | Target | 18 Aug gate? |
|-----------|--------|----------------|
| Mock login, `/me`, HomeFeed, Continue Watching | Artificial latency **~400–600 ms** so loading/error paths are real (DF2), not instant | No |
| Cold-start / post-login first paint | Shell loader until `/me` + HomeFeed + CW complete; perceived wait **&lt; ~2 s** on a release build | No — if slower, release build |
| Credentials inline error | Same mock RTT; no extra round-trip; not an alert | Yes as UX |
| Vertical `loadMore` (next `Container[]` page) and horizontal `loadMore` (`resources` / cards) | Virtualized lists; fetch in the background; no scroll jank as a *program*, only as craft | Qualitative |
| Silent Continue Watching reload | Stale-while-revalidate on the `progress` container; **no** full-screen loader | Yes as UX |

**Already required (doc 15):** Hermes on; virtualized lists; placeholder art only at target
aspect ratios; no background fetch, video SDK, or push/Firebase native blobs.

---

## 3. Stateful vs. stateless

A phone is not a cluster. What matters is **what the device remembers** vs **what is a stand-in
for the future API**.

| Kind | What | Where |
|------|------|--------|
| **Stateless** | Shared kernel (theme / UI / i18n / analytics), API-client functions, screens | No catalog cache of their own |
| **Stateful on device (normal)** | Auth slice (JWT + `exp`) | Keychain via redux-persist |
| | User slice, content slice (`Container[]` + pagination cursors), NetInfo, navigation | Memory |
| **Stand-in “DB”** | Fixtures + mock adapter | API module, in-process. **Removed in Phase 3** |
| **Does not exist** | Server sessions, Redis, job queues, SQLite / MMKV | — |

The device remembers **login**. **Catalog is refetchable.** Mocks are a fake backend, not a
client cache of record. Screens and hooks are not data owners (docs 03–04).

---

## 4. Bottlenecks & scaling strategy

**First bottleneck under growth:** the **storefront on-device** — JS thread, image decode, and
a content slice that **only grows** on `loadMore` (no eviction). Not a database, Redis, a Node
process, or a third-party rate limit (none of those exist here). The mock adapter is not a
bottleneck; it is replaced.

**Phase 3 (not ours):** payload size and artwork CDN for HomeFeed / CW. Do not pre-empt that
with SQLite or a mock HTTP server.

**Scaling strategy**

| What | Strategy |
|------|----------|
| **Phase 1 (this repo)** | **Do not scale infra.** One process per device. “Bigger box” = **release build + Hermes** |
| **More users** | Already **horizontal**: N devices, N stores. JWT on the request; no sticky server session |
| **Phase 3 API (backend team)** | They choose vertical or horizontal. We stay **stateless on the wire**: axios instance + auth interceptor, swappable base URL |
| **Independent axes** | Auth vs storefront modules; vertical container pages vs horizontal `resources` pages; mock → HTTP is a swap, not a cluster |

---

## 5. Caching & async

| Data | Cache | Invalidation |
|------|--------|----------------|
| JWT | Yes — Keychain | Logout, `/me` 401, `exp` |
| HomeFeed `Container[]` (hero + other rows) | **RAM** only (content slice). No disk | Vertical `loadMore`; logout |
| Continue Watching `Container[]` (`variant: "progress"`) | **RAM**, stale-while-revalidate | Silent refetch when the storefront screen is shown again; logout |
| `resources: Card[]` | Travel **with** their container. No global Card store | Horizontal page of that container; logout |
| Artwork | App bundle + React Native `Image` cache. No Fast Image / extra disk cache | Next binary |
| HTTP cache / React Query / CDN client | **None** | — |

**Async / queues:** none. Mocks are Promises with latency. No background fetch, no prefetch of
every container, no WorkManager / BGTask. The only deferred work is `loadMore` at list
end-reached.

The shell loader **does** block first paint until the three boot calls complete (visual
fidelity; no layout jump when CW inserts under hero).

Phase 3 may add image prefetch or a queue **behind the same storefront hooks** without
changing screens.

---

## 6. Don't-foreclose

| Phase-1 shortcut | Blocks horizontal scaling later? | Cheap mitigation now |
|------------------|----------------------------------|----------------------|
| In-process mock as “DB” | No — dropped in Phase 3 | I/O only through the API module (DF1) |
| Catalog / `Container[]` RAM-only | No (each device is a replica) | Do not persist content; refetch |
| No eviction as the user pages | No for servers; **yes** for on-device memory on a long catalog | Pagination + virtualized lists now; windowing later if needed |
| JWT in Keychain, no server session store | No | Header interceptor; no sticky host |
| Boot: three calls before first paint | No | Parallel; do not add more to the gate |
| Client inserts CW under hero | No | Two GETs; silent reload replaces only the `progress` container |
| Extra `Card` fields TBD | No | Additive optionals; 1.11 names JSON |
| No queues / prefetch / Fast Image | No | `loadMore` behind hooks; prefetch later without touching screens |
| Single RN process (modular monolith) | N/A — nothing to split into a cluster | Features do not import each other (DF5) |
| Mock latency 400–600 ms | No | Goes away with the mock |
| No FPS/size SLO; release build if jank | No | Hermes + virtualization (doc 15) |

Do **not** add Redis, a mock HTTP server, SQLite, or sticky sessions “just in case.”

---

## 7. Storefront payload (scale-relevant shape)

Locked in this session so cache, paging, and component reuse share one type. Living field
list: `architecture/04-data-model.md`. Wire JSON names: session 1.11.

**Shared types (both feed GETs):**

- **`Container`** — row: own `name` and container metadata, `variant`, **`resources: Card[]`**.
- **`Card`** — tile: content **name** plus other content fields **TBD** (Phase 1 mocks fill
  what the UI already needs; production extras stay open).

**HomeFeed (JWT):** a **`Container[]`**. The spotlight is a container with
`variant: "hero"`. First page remains **one hero + 15 other containers**. Further vertical
pages are more containers (no second hero).

**Continue Watching (JWT, separate GET — per-user):** an attribute of the **same**
`Container[]` type. Typically **one** container, `variant: "progress"` (timeline chrome).
`resources` are the cards for that row.

**Client composition** (unchanged split, new envelope):

```
[ HomeFeed hero container ] + [ CW progress container ] + [ remaining HomeFeed containers ]
```

Reuse one **Container** component and one **Card** component; variant selects chrome
(`hero`, `progress`, portrait/landscape; `live` later). Continue watching is **not** mixed
into the HomeFeed **response** — only into the composed list — so silent reload can replace
the `progress` container alone.

ADR-0006 (two GETs) still holds; ADR-0007 records the shared types. Envelope field names
for the `Container[]` attribute remain 1.11 (OQ-22).

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Load now | **&lt;10 concurrent sessions**, in-process mocks, ~16 HomeFeed containers + CW | Matches the stakeholder demo | Sizing a server we do not build |
| 2 | Load later / 100× | Production streaming app: **thousands of clients**, real catalog/CDN; still no backend here | Honest ceiling without over-building | Treating Phase 1 as public scale |
| 3 | Performance targets | UX targets only (mock **400–600 ms**, first paint **&lt; ~2 s** perceived); **no** FPS/size/p99 SLOs as 18 Aug gates | Docs 01/15; release build if jank | Load-test / perf program for the demo |
| 4 | State | Persist **auth only**; catalog/user **memory**; mocks are the fake DB | Device-local; Phase 3 swap | Persisting HomeFeed; SQLite; server session store |
| 5 | First bottleneck | **On-device storefront** (lists, images, growing RAM slice) | Only teething we own | Inventing a DB/queue bottleneck |
| 6 | Scale strategy | **No infra scale** in Phase 1; devices already horizontal; Phase 3 backend scales behind the API module | Nothing to cluster | Sticky hosts; a Phase-1 mock HTTP process |
| 7 | Cache & async | Token + RAM slices; CW stale-while-revalidate; **no** disk catalog, queues, or prefetch | Enough for &lt;10 sessions | Fast Image / React Query / background fetch in Phase 1 |
| 8 | Feed shape | Both GETs return **`Container[]`** with **`resources: Card[]`**; hero / progress are **variants**; CW still a **separate** JWT API | One Container/Card component; silent CW reload stays a small call | `{ hero, carousels }` envelope; CW mixed into HomeFeed; distinct CW-only components |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-22 | JSON names and paths for `/me`, HomeFeed, Continue Watching (the `Container[]` attribute), page envelope (`nextCursor`), Container, and Card | Mobile | 1.11 Interface Contracts |
| OQ-23 | Cards per container horizontal page | Mobile | 1.11; storefront STEP |
| OQ-26 | Remaining **Card** fields beyond content name (artwork keys, rating, progress/timeline fields, etc.) and which are `progress`-only | Mobile | 1.11; 1.7 for chrome |
| OQ-27 | When (if ever) to evict old container/card pages from the content slice | Mobile | Revisit if a long paginated feed is real (Phase 3+) |

Carried forward: OQ-17 (mock strategy → 1.11), OQ-24 (hero chrome vs stand-in → 1.7).

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.5 | Initial draft from the scaling & performance session |
