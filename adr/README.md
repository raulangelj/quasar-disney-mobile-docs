# Architecture Decision Records

Point-in-time records of **why quasar-disney-mobile is the way it is**. Each ADR captures one
decision: the context, the choice, the alternatives rejected, and the consequences. See
`METHOD.md` §3.

## Conventions
- Filenames: `ADR-NNNN-kebab-title.md`, sequential, zero-padded, never reused.
- **Reserve the number like a STEP number** (teams): the registry below is shared, so two
  authors can each append `ADR-0005` and git will merge both without a conflict — a silent
  duplicate. Pull, take `max + 1` over the registry, add your row and create the ADR file, then
  **commit and push immediately**. Before every push, even on a clean merge, scan for a repeated
  number (`grep -oE '^\|[[:space:]]*ADR-[0-9]+' adr/README.md | grep -oE 'ADR-[0-9]+' | sort | uniq -d`); if it's non-empty or the push
  is rejected, recompute `max + 1`, renumber, and push again. Solo, it's just a local edit. See
  `../runbooks/collaboration.md` §6.
- Write decisions a future contributor would ask "why did they do it this way?" about —
  not tactical choices (library versions, file layout).
- **Never rewrite an accepted decision.** If it changes, append a dated `## Amendment`,
  or write a new ADR and set this one's status to `Superseded by ADR-XXXX`.
- Use `../templates/adr-template.md`. Add every new ADR to the registry below.

## How decisions get accepted
ADRs are how a decision is socialized — especially in a team (see
`../runbooks/collaboration.md`).
- **Solo:** author straight to **Accepted** — you're the authority.
- **Team:** a significant decision lands as **Proposed**, gets reviewed, then is flipped to
  **Accepted** by the designated authority. Re-running a session that changes a decision
  others depend on follows the same flow: write the new ADR **Proposed**, review, **Accept**,
  and mark the old one `Superseded by ADR-XXXX` — never revise a shared decision silently.

**Who accepts an ADR in this project:** _solo author_ <!-- record your rule when a team forms:
e.g. "tech lead", "consensus of maintainers", "ADR review on PR". -->

## Registry

> First column = the full `ADR-NNNN` id (e.g. `ADR-0001`), **not** a bare number — the
> duplicate-scan above keys on that `ADR-` prefix, so a bare number makes it silently miss
> collisions.

| ADR | Title | Status | Date |
|------|-------|--------|------|
| ADR-0001 | Modular monolith with a composition-root shell | Accepted | 2026-08-16 |
| ADR-0002 | In-app API module with axios; no backend component | Accepted | 2026-08-16 |
| ADR-0003 | Persist the auth slice to platform secure storage from day one | Accepted | 2026-08-16 |
| ADR-0004 | Online-only client with a shell-level connectivity gate | Accepted | 2026-08-16 |
| ADR-0005 | Storefront pagination owned by feature hooks | Accepted | 2026-08-16 |
| ADR-0006 | Two authenticated storefront GETs composed on the client | Accepted | 2026-08-16 |
| ADR-0007 | Shared Container and Card types on both storefront GETs | Accepted | 2026-08-17 |
| ADR-0008 | Abbreviated Phase-1 threat model with recorded deferrals | Accepted | 2026-08-17 |
| ADR-0009 | Phase-1 mock auth; Phase-3 managed IdP behind our API | Accepted | 2026-08-17 |
| ADR-0010 | Binary session gate; no roles or entitlements in Phase 1 | Accepted | 2026-08-17 |
| ADR-0011 | Theme is brand × surface mode, and modes are named by role | Accepted | 2026-08-17 |
| ADR-0012 | Bundle Inter rather than use the platform system font stack | Accepted | 2026-08-17 |
| ADR-0013 | Adopt `react-native-svg`, and complete doc 03's dependency list | Accepted | 2026-08-17 |
| ADR-0014 | The connectivity gate keys on interface state, not internet reachability | Accepted | 2026-08-17 |
| ADR-0015 | Config reaches the app through a Babel transform, not native build config | Accepted | 2026-08-17 |

<!-- Example row shape (indented so it isn't picked up by the scan / `max + 1`; a real row
     starts at the line's left margin with no leading spaces):
       | ADR-0001 | Use Postgres as the primary datastore | Accepted | 2026-01-15 |
-->
