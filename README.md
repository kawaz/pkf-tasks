# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Drop-in jj/git auto-dispatch, doc-integrity, SemVer gates, and upstream-drift detection for `kawaz/*` projects.

> This is a shared task collection for **kawaz's own workflow** across `kawaz/*` repos, not a public reference implementation (fork / borrow freely). Naming and granularity are optimised for keeping `kawaz/*` repos consistent with each other.

## Modules

- [vcs](./tasks/vcs/) — jj/git auto-dispatched VCS primitives + knowledge storage
- [docs](./tasks/docs/) — translation-pair (`*-ja.md` / `*.md`) integrity check (v2.1.0+ split into commit-lag / links + acceptsArgs)
- [semver](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI required)
- [migrate](./tasks/migrate/) — upstream-drift detection + auto-fix for pkf-tasks / pkfire (`bump-semver` CLI required)

Each module directory has its own README with task details, parameters, and usage examples.

## Usage

### Bundled import (one-liner via `all.pkl`)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterise per-instance
}
```

### Per-module import (pick only what you need)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/docs.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases).

### Using `docs:check-translations` (v2.1.0+)

With `acceptsArgs = true`, source files / globs can be passed via CLI. For `-ja.md` style sources, neighboring `*.md` files are discovered as translation targets:

```bash
# auto-discover (every *-ja.md under cwd as source)
pkf run docs:check-translations

# explicit list (kawaz convention)
pkf run docs:check-translations -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# wide glob
pkf run docs:check-translations -- '**/*-ja.md'

# individual sub-checks
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'   # only the commit-lag sub-check (skip links)
pkf run docs:check-translation-links -- README-ja.md         # only the links sub-check (skip commit-lag)
```

When the source is `*.md` (e.g. English-original projects), `*[_.-]*.md` (`-ja.md` / `-zh.md` etc.) are scanned, supporting multilingual setups too. The link check is bilingual-only (sources with 3+ translation targets log a skip and pass).

## SemVer stability (2.x)

pkf-tasks follows SemVer 2.0.0 from v1.0.0 onward. The **public API contract** (no breaking changes within the 2.x line) covers:

- Group entry FQNs (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `semver.pkl` / `migrate.pkl`) and their exported field names
- The bundled entry `tasks/all.pkl` and the `kawaz.<group>.<field>` namespace it exposes
- Public task names (`vcs:commit`, `docs:check-translations`, `docs:check-translation-commit-lag`, `docs:check-translation-links`, `semver:compare`, `migrate:check-pkf-tasks-current`, etc.)

Internal implementation files (`vcs/iface.pkl`, `vcs/jj.pkl`, `vcs/git.pkl`, `vcs/auto.pkl`, `docs/translations.pkl`, `semver/check-bumped.pkl`, `migrate/check-current.pkl`, etc.) are **not** part of the public contract — they may move or be renamed in any minor/patch release. Always import via the flat group entries.

Pin to `pkf-tasks@2` for major-only floating, or to an exact version (e.g. `pkf-tasks@2.1.0`) for full reproducibility. The `migrate:check-*` gates keep the pin up to date regardless.

Pkfire 0.6.0+ supports **glob targets** on the CLI, composing well with the `<group>:<action>` task naming:

```bash
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
