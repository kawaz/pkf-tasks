# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Groups provided:

- **`vcs/`** — jj/git auto-dispatched task primitives (`commit` / `push` / `fetch` / `ensureClean` / `fetch-tags`) plus Pkl function helpers (`diffSummary` / `readAtRef`) for embedding inside other Task `cmd`s. Acts as the centralised store for jj/git operational knowledge (DR-0006).
- **`docs/translations.pkl`** — translation-pair integrity check (paired `*-ja.md` / `*.md` documents — existence, mutual link, commit-timestamp order).
- **`lint/`** — `lint:pkl` (`pkl format -w`) and `lint:all-coverage` (orphan-module detection across the package itself).
- **`semver/`** — parameterised SemVer gates (`check-bumped` for push-time / release-time VERSION bump enforcement) and an ad-hoc `compare` task wrapping `bump-semver compare`. Requires `bump-semver` CLI on PATH.
- **`migrate/`** — `migrate:check-pkf-tasks-current` (gate, fails if Taskfile.pkl's pkf-tasks import is behind latest release) and `migrate:update-pkf-tasks` (action, sed-rewrites the import to the latest tag). Wire `migrate:check` into `push`'s deps to surface upgrades at the right moment.

## Usage

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.13#/all.pkl" as kawaz

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

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases). Pin to an exact version while in the 0.0.x phase; `migrate:check-pkf-tasks-current` keeps the pin up to date.

More: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN.md](./docs/DESIGN.md) / [docs/decisions/](./docs/decisions/) / example consumer [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
