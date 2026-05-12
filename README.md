# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Drop-in jj/git auto-dispatch, doc-integrity, lint, SemVer gates, and upstream-drift detection for `kawaz/*` projects.

## Modules

- [`tasks/vcs/`](./tasks/vcs/) — jj/git auto-dispatched VCS primitives + knowledge storage
- [`tasks/docs/`](./tasks/docs/) — translation-pair (`*-ja.md` / `*.md`) integrity check
- [`tasks/lint/`](./tasks/lint/) — language-agnostic Pkl format + orphan-module detection
- [`tasks/semver/`](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI required)
- [`tasks/migrate/`](./tasks/migrate/) — upstream-drift detection + auto-fix for pkf-tasks / pkfire (`bump-semver` CLI required)

Each module directory has its own README with task details, parameters, and usage examples.

## Usage

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.17#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterise per-instance
}
```

The `all.pkl` aggregation endpoint exposes every group through a single `kawaz` namespace. Per-module imports (`import ".../vcs/auto.pkl" as vcs`) also work for finer control.

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases). Pin to an exact version while in the 0.0.x phase; the `migrate:check-*` gates keep the pin up to date.

Pkfire 0.6.0+ supports **glob targets** on the CLI, composing well with the `<group>:<action>` task naming:

```bash
pkf run 'lint:*'            # all lint:* tasks in topological order
pkf run 'semver:*'          # all semver: gates / utilities at once
pkf list 'migrate:check-*'  # inspect just the check-* migration gates
```

More: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN.md](./docs/DESIGN.md) / [docs/decisions/](./docs/decisions/) / example consumer [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
