# Changelog

All notable changes to `kawaz/pkf-tasks` are recorded here. The package is in **0.0.x unstable phase** — every release may contain breaking interface changes; pin to an exact version (`@0.0.5`, not `@0`) until 0.1.0 stabilises the surface.

## 0.0.5 — 2026-05-11

### Breaking

- **`tasks/vcs/jj.pkl` / `tasks/vcs/git.pkl`**: the `env { ["SSH_AUTH_SOCK"] = read("env:SSH_AUTH_SOCK") }` declaration on `push` / `fetch` tasks was **removed**. `read("env:...")` raises `Cannot find resource` during Pkl evaluation when the variable is unset, which made the modules un-evaluable in environments without an active SSH agent (CI runners, fresh shells). The `SSH_AUTH_SOCK` is now inherited from the host via pkfire's `inheritEnv = true` default, so callers no longer need to set it explicitly. Migration: nothing to do on the caller side; signing still works locally because the host shell exports `SSH_AUTH_SOCK`. CI environments that previously failed during `pkf list` / `pkl eval` will now succeed.

### Improved

- `tasks/vcs/auto.pkl`: `description` of each auto-selected task is now rewritten to "vcs commit/push/fetch/... (auto-detect: jj or git)" so callers do not see jj-specific wording on the dispatcher.
- `tasks/vcs/auto.pkl`: `indent` helper no longer emits trailing two-space lines for empty input rows (defensive improvement against future cmd edits).
- `README.md` / `README-ja.md`: corrected "amends" → "extends" mismatch (abstract modules in Pkl must be extended, not amended). Usage example version updated to `pkf-tasks@0.0.5`.

## 0.0.4 — 2026-05-11

### Fixed

- **`tasks/docs/translations.pkl`**: the bash `for` loop in the `docs:check-translations` task **did not propagate failures**. When `check_pair` failed inside the loop (e.g. translation timestamp out of order), bash continued to the next iteration and the loop's overall exit code was the last iteration's return value. Since `docs/MANUAL` typically has no `-ja.md` and `check_pair` returns 0 (skip), the gate **silently passed even when README or DESIGN pairs failed**. Fixed by adding `set -euo pipefail` and an explicit `failed=0` accumulator with `exit $failed` at the end. Affects every consumer that uses `docs.checkTranslations` (notably `kawaz/bump-semver`) — the gate is now actually enforced.

## 0.0.3 — 2026-05-11

### Fixed

- **`tasks/vcs/auto.pkl` / `tasks/docs/translations.pkl`**: VCS detection switched from `[ -d .jj ]` / `[ -d .git ]` to `jj root >/dev/null 2>&1` / `git rev-parse --git-dir >/dev/null 2>&1`. The old pattern only inspected the current directory and produced false negatives inside `jj workspace add`-style subdirectories where `.jj` lives at the workspace root (not under cwd). The new pattern uses the upstream CLIs' built-in upward search.

## 0.0.2 — 2026-05-11

### Breaking

- **`tasks/vcs/iface.pkl`**: `allTasks: Listing<Task>` was changed from `local` (module-private) to public so callers can write `tasks { ...vcs.allTasks }`. In 0.0.1 the property existed but was not externally accessible.

## 0.0.1 — 2026-05-11

### Added

- Initial release. Provides:
  - `tasks/vcs/{iface,jj,git,auto}.pkl`: abstract module + value-level selection between jj and git, dispatched at runtime inside cmd by probing the filesystem.
  - `tasks/docs/translations.pkl`: integrity check task for paired `*-ja.md` / `*.md` documents (existence, mutual link, commit-timestamp order).
- Distributed via GitHub Releases → `pkg.pkl-lang.org` proxy, with SHA256-verified zip + metadata.
