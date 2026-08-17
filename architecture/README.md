# Architecture

The living, canonical description of **what quasar-disney-mobile is**. These docs are the source of
truth for the current design — kept up to date as the system evolves (unlike ADRs, which
are point-in-time). See `METHOD.md` §3 and §6.

Most of these are produced by the **architecture sessions** during STEP-1
(`../templates/architecture-sessions/`). To create or revise one, run its session
(e.g. *"Run STEP-1.5: Scaling & Performance"*).

## Conventions
- Filenames: `NN-kebab-title.md`, numbered in the order the sessions produce them.
- Every doc carries: `Version` (major.minor.patch), `Status` (Draft → Current → Deprecated),
  `Last updated`, `Audience`, a **Decision Summary**, **Open Questions**, and a
  **Version Log**. Use `../templates/architecture-doc-template.md`.

## Index

> Filled in and reconciled by the **Cross-Cutting Review** once the docs exist, and kept
> current at each **check-in** (`../runbooks/check-in.md`). Each doc's own
> header is the authoritative Version/Status; this table is a convenience map. During STEP-1,
> `prompts/STEP-index.md` is the live view of which sessions have produced their docs.

| # | Doc | Version | Status |
|---|-----|---------|--------|
| 01 | [System Overview, Requirements & Non-Goals](01-system-overview.md) | v0.1.4 | Draft |
| 02 | [Phasing & Roadmap](02-phasing-roadmap.md) | v0.3.3 | Draft |
| 03 | [Architecture Overview & Component Boundaries](03-architecture-overview.md) | v0.3.2 | Draft |
| 04 | [Data Model, Ownership & Retention](04-data-model.md) | v0.2.2 | Draft |
| 05 | [Scaling & Performance](05-scaling-performance.md) | v0.1.0 | Draft |
| 06 | [Security & Threat Model](06-security-threat-model.md) | v0.1.1 | Draft |
| 15 | [Native App Architecture](15-native-app-architecture.md) | v0.2.3 | Draft |
| 16 | [Identity & Auth](16-identity-auth.md) | v0.1.0 | Draft |
| _(remaining core-block rows filled in by later sessions / the Cross-Cutting Review)_ | | | |
