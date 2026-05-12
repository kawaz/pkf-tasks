# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Drop-in jj/git auto-dispatch, doc-integrity, lint, SemVer gates, and upstream-drift detection for `kawaz/*` projects.

## Modules

- [vcs](./tasks/vcs/) — jj/git auto-dispatched VCS primitives + knowledge storage
- [docs](./tasks/docs/) — translation-pair (`*-ja.md` / `*.md`) integrity check
- [lint](./tasks/lint/) — language-agnostic Pkl format + orphan-module detection
- [semver](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI required)
- [migrate](./tasks/migrate/) — upstream-drift detection + auto-fix for pkf-tasks / pkfire (`bump-semver` CLI required)

Each module directory has its own README with task details, parameters, and usage examples.

## Usage

### Bundled import (one-liner via `all.pkl`)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterise per-instance
}
```

### Per-module import (pick only what you need)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/docs.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases).

## SemVer stability (1.x)

pkf-tasks follows SemVer 2.0.0 from v1.0.0 onward. The **public API contract** (no breaking changes within the 1.x line) covers:

- Group entry FQNs (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `lint.pkl` / `semver.pkl` / `migrate.pkl`) and their exported field names
- The bundled entry `tasks/all.pkl` and the `kawaz.<group>.<field>` namespace it exposes
- Public task names (`vcs:commit`, `docs:check-translations`, `lint:pkl`, `semver:compare`, `migrate:check-pkf-tasks-current`, etc.)

Internal implementation files (`vcs/iface.pkl`, `vcs/jj.pkl`, `vcs/git.pkl`, `vcs/auto.pkl`, `docs/translations.pkl`, `lint/pkl.pkl`, `semver/check-bumped.pkl`, `migrate/check-current.pkl`, etc.) are **not** part of the public contract — they may move or be renamed in any minor/patch release. Always import via the flat group entries.

Pin to `pkf-tasks@1` for major-only floating, or to an exact version (e.g. `pkf-tasks@1.0.0`) for full reproducibility. The `migrate:check-*` gates keep the pin up to date regardless.

Pkfire 0.6.0+ supports **glob targets** on the CLI, composing well with the `<group>:<action>` task naming:

```bash
pkf run 'lint:*'            # all lint:* tasks in topological order
pkf run 'semver:*'          # all semver: gates / utilities at once
pkf list 'migrate:check-*'  # inspect just the check-* migration gates
```

## More

- [CHANGELOG](./CHANGELOG.md)
- [docs/DESIGN.md](./docs/DESIGN.md) — internal design notes
- [docs/decisions/](./docs/decisions/) — Decision Records (DR)
- Example consumer: [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
