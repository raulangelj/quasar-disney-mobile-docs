# Shell (Bash) coding standards — quasar-disney-mobile

> **Reconciled to this project in STEP-1.12** (`architecture/12-test-strategy.md` §10.3). Kept
> because the docs hub ships `scripts/*.sh` and the root `doctor.sh`, and doc 09 §7.2 contemplates a
> small `.env` key-completeness script. Otherwise close to the shipped default.
>
> **Cross-cutting.** Shell here is CI glue and developer scripts. **There are no containers and no
> entrypoints** — doc 08 §1 hosts nothing.

**Baseline:** this standard targets **bash** — it is bash-specific, not portable POSIX `sh`.
Reach for `sh`/`dash` only when a constraint requires it, and say so at the top of the script.
Follow the [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html); lint
with **ShellCheck** and format with **shfmt**.

**ShellCheck and shfmt are *not* a Phase-1 CI gate.** The merge gate is a JS-only job — `tsc`,
Jest, ESLint, Prettier (doc 12 §7, ADR-0018) — and the only shell this project authors in Phase 1a is
possibly the `.env` completeness check. Run them locally or from a pre-commit hook; revisit adding
them to CI when Bitrise lands (Phase 2, OQ-05). Saying they are "pinned in CI" when they are not is
how a standards doc stops being trusted.

**Prefer a real language past simple scripts (recommendation).** Shell is best for small
utilities and wrappers. When a script grows beyond ~100 lines, or needs non-straightforward control
flow or real data structures, move it to **TypeScript under the Node tooling runtime** (doc 12 §7) or
split it — this stack has no Python or Go to fall back on. A guideline to weigh, not a hard gate.

## Preamble & strict mode
- **Shebang: `#!/usr/bin/env bash`** — settled for this project. It resolves bash via `PATH`, which
  matters on the macOS development machines, where `/bin/bash` is still 3.2 and a modern bash comes
  from Homebrew. All existing scripts (`scripts/*.sh`, root `doctor.sh`) already use it; this records
  the convention rather than introducing one.
- Start scripts with **strict mode**: `set -euo pipefail` — exit on error, error on unset
  variables, and fail a pipeline if any stage fails. Be aware `set -e` has well-known edge cases,
  so still check explicitly any command whose failure you must handle. Set `IFS=$'\n\t'` when
  word-splitting behavior matters.

## Safety & quoting
- **Always quote expansions** — `"${var}"`, `"$(cmd)"`, and `"$@"` (never `$*`). Unquoted
  expansion causes word-splitting and glob bugs, the single most common shell defect; ShellCheck
  flags them.
- Use **arrays** for argument lists and iterate with `"${arr[@]}"`. Avoid **`eval`**. Use explicit
  paths with globs (`./*`, not `*`), and don't parse the output of `ls`.

## Naming & layout
- `lower_snake_case` for variables and functions; `UPPER_SNAKE_CASE` for constants and exported
  environment (`readonly MAX_RETRIES=3`). Declare function-local variables with **`local`**.
- 2-space indentation (no tabs), lines ≤ 80 columns. Put `; then` / `; do` on the same line as
  `if`/`for`/`while`, and `{` on the same line as the function name. Define functions near the top;
  if a script has multiple functions, put the body in **`main()`** and call `main "$@"` at the end.

## Commands & idioms
- Prefer **`[[ … ]]`** over `[ … ]`/`test`; use `==` for string comparison and **`(( … ))`** for
  arithmetic and numeric comparison (never `<`/`>` inside `[[ … ]]`). Use **`$(…)`**, not
  backticks.
- Prefer shell builtins over spawning external processes. Avoid aliases in scripts — use functions.

## Error handling
- Send **all error and diagnostic messages to STDERR** (`echo "..." >&2`); keep STDOUT for real
  output. Return meaningful **exit codes** (`0` success, non-zero failure).
- Use a **`trap … EXIT`** to clean up temp files and resources even on failure
  (`trap 'rm -f "${tmpfile}"' EXIT`). Check return values — `if cmd; then …` or `$?` — and inspect
  individual stages of a pipeline via the **`PIPESTATUS`** array.
