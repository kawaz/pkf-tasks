# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Groups provided:

- **`vcs/`** — jj/git auto-dispatched VCS primitives + knowledge storage (DR-0006).
  - Tasks: `vcs:commit` / `vcs:push` / `vcs:fetch` / `vcs:ensure-clean` / `vcs:fetch-tags`
  - Pkl function helpers (for embedding inside other Task `cmd`s): `vcs.diffSummary(ref, paths)` / `vcs.readAtRef(ref, path)`

- **`docs/translations.pkl`** — translation-pair integrity check.
  - Task: `docs:check-translations` — verifies paired `*-ja.md` / `*.md` documents (existence, mutual link, commit-timestamp order)

- **`lint/`** — language-agnostic + meta lint.
  - `lint:pkl` — `pkl format -w` over `**/*.pkl` + `PklProject*`
  - `lint:all-coverage` — orphan-module detection across the package itself

- **`semver/`** — parameterised SemVer gates and an ad-hoc compare. Requires `bump-semver` CLI on PATH.
  - `semver:check-bumped` (parameterise per-instance via object-amends) — fail if `triggerPaths` changed since `compareRef` without a corresponding VERSION bump
  - `semver:compare` — `acceptsArgs = true` wrapper around `bump-semver compare` (ad-hoc CLI use, e.g. `pkf run semver:compare -- gt VERSION 1.0.0`)

- **`migrate/`** — version-drift gates + auto-fix actions for the consumer's `Taskfile.pkl`. Two upstream targets, two pairs each (check / update).
  - **pkf-tasks** (`import` URI):
    - `migrate:check-pkf-tasks-current` — fail if Taskfile.pkl's `pkf-tasks@<version>` import is behind latest release
    - `migrate:update-pkf-tasks` — sed-rewrite the import to the latest tag
  - **pkfire** (`amends` URI, v0.0.15+):
    - `migrate:check-pkfire-current` — fail if Taskfile.pkl's `pkfire@<version>` amends is behind latest release
    - `migrate:update-pkfire` — wraps `pkf migrate --to=<latest>` (pkfire 0.6.0+ built-in, eval-verified, auto-reverts on failure)
  - Wire the `check-*` tasks into `push`'s `deps` to surface upgrades at the right moment.

## Tasks at a glance

Usefulness legend: ★★★ = required across most kawaz repos / wired into `push` deps / ★★ = frequently used or essential for specific use cases / ★ = ad-hoc / special-purpose / internal-implementation

### `vcs/` — jj/git auto-dispatched VCS primitives + knowledge storage

| Task | Usefulness | Purpose | Args / Notes |
|---|---|---|---|
| `vcs:commit` | ★★★ | Commit working-copy changes if any (jj/git auto-dispatch) | param: `message` (required) |
| `vcs:push` | ★★★ | Push to remote (jj/git auto-dispatch) | Extend with `(vcs.push) { deps { ... } }` to wire pre-push gates |
| `vcs:fetch` | ★★ | Fetch from remote (jj/git auto-dispatch) | — |
| `vcs:ensure-clean` | ★★★ | Verify working copy is clean (jj/git auto-dispatch) | Wire into `push` gates |
| `vcs:fetch-tags` | ★★ | Sync tags (jj/git auto-dispatch) | Precedes `semver:check-against-latest-release` etc. |

> Note: there are also Pkl helper functions (`vcs.diffSummary` / `vcs.readAtRef`) that return bash `$(...)` fragments for string-interpolation into other Tasks' cmds. They are internal-implementation / advanced use; see [DESIGN.md](./docs/DESIGN.md).

### `docs/` — translation-pair integrity

| Task | Usefulness | Purpose | Args / Notes |
|---|---|---|---|
| `docs:check-translations` | ★★★ | Translation-pair (`*-ja.md` / `*.md`) integrity check | Wire into `push` deps |

### `lint/` — language-agnostic + meta lint

| Task | Usefulness | Purpose | Args / Notes |
|---|---|---|---|
| `lint:pkl` | ★★★ | Auto-format all Pkl files (`pkl format -w`) | Wire into `push` deps |

> An internal `lint:all-coverage` (orphan-module detection) also exists; rarely used by consumers.

### `semver/` — SemVer gates + ad-hoc compare (`bump-semver` CLI required)

| Task | Usefulness | Purpose | Args / Notes |
|---|---|---|---|
| `semver:check-bumped` | ★★★ | Version-bump gate (fail if version files weren't bumped after relevant changes) | Parameterise per-instance via object-amends; usage example below |
| `semver:compare` | ★★ | Thin wrapper around `bump-semver compare` | e.g. `pkf run semver:compare -- gt VERSION 1.0.0` |

Usage example for `semver:check-bumped`:

```pkl
(kawaz.semver.checkBumped) {
  compareRefCmd = "echo main@origin"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")  // consumer project's version file
  taskName = "semver:check-version-bumped"
}.check
```

### `migrate/` — upstream-drift pairs (`bump-semver` CLI required)

| Task | Usefulness | Purpose | Args / Notes |
|---|---|---|---|
| **`migrate:check-*`** | — (glob, gate group) | Detect upstream drift | `pkf run 'migrate:check-*'` to run all; wire into `push` deps |
| └ `migrate:check-pkf-tasks-current` | ★★★ | Fail if pkf-tasks `import` is behind latest release | — |
| └ `migrate:check-pkfire-current` | ★★★ | Fail if pkfire `amends` is behind latest release | — |
| **`migrate:update-*`** | — (glob, fix group) | Auto-fix upstream drift | `pkf run 'migrate:update-*'` to run all; remediation when checks fail |
| └ `migrate:update-pkf-tasks` | ★★ | Rewrite pkf-tasks `import` to latest tag | No auto-commit |
| └ `migrate:update-pkfire` | ★★ | Rewrite pkfire `amends` to latest tag (eval-verified) | No auto-commit |

## Usage

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.15#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterise per-instance
}
```

The `all.pkl` aggregation endpoint (v0.0.10+) exposes every group through a single `kawaz` namespace. Per-module imports (`import ".../vcs/auto.pkl" as vcs`) still work for finer control.

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases). Pin to an exact version while in the 0.0.x phase; the `migrate:check-*` gates keep the pin up to date.

Pkfire 0.6.0+ supports **glob targets** (`path.Match` syntax) on the CLI, which composes well with the `<group>:<action>` task naming:

```bash
pkf run 'lint:*'            # all lint:* tasks in topological order
pkf run 'semver:*'          # every semver: gate / utility at once
pkf list 'migrate:check-*'  # inspect just the check-* migration gates
```

More: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN.md](./docs/DESIGN.md) / [docs/decisions/](./docs/decisions/) / example consumer [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
