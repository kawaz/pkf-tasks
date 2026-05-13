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
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.7.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.2#/all.pkl" as kawaz

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
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.7.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.2#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.2#/docs.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases).

### Using `docs:check-translations` (v2.1.0+)

With `acceptsArgs = true`, source files / globs can be passed via CLI. The sub-checks (`commit-lag` / `links`) each accept the same arguments. The umbrella `docs:check-translations` does **not** forward args to its deps (pkfire orchestrator limitation in 0.6.0), so use the sub-checks directly when passing CLI args:

```bash
# auto-discover (every *-ja.md under cwd; excludes .jj/ .git/ .out/ node_modules/)
pkf run docs:check-translations

# explicit list (kawaz convention)
pkf run docs:check-translation-commit-lag -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# wide glob (bash 4+ for **; bash 3.2 expands ** as * — use brace expansion {,*/,*/*/}*-ja.md instead)
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'
pkf run docs:check-translation-links -- '**/*-ja.md'
```

#### Neighbor discovery (source → translation targets)

For each source file, neighboring files with matching basename are discovered as translation targets (the source itself is excluded):

| source pattern | targets discovered | use case |
|---|---|---|
| `<base>-ja.md` (kawaz convention) | `<base>.md` only (1:1, en) | ja-original projects |
| `<base>.md` (e.g. English-original) | `<base>-??.md` / `<base>-???.md` (2-3 letter language codes: `-ja.md`, `-zh.md`, `-jpn.md`, etc.) | en-original / multilingual |

The `-ja.md` source uses 1:1 discovery to avoid false positives from siblings like `data-layout-history-ja.md` being treated as a translation of `data-layout-ja.md`. The `*.md` source restricts to short language-code suffixes for the same reason.

The link check only validates the bilingual ja ↔ en convention (1 source ↔ 1 target). Sources with 0 targets (no translation yet) or 2+ targets (multilingual) log a skip and pass — the link convention hasn't been generalised for those cases.

## SemVer stability (2.x)

pkf-tasks follows SemVer 2.0.0 from v1.0.0 onward. The **public API contract** (no breaking changes within the 2.x line) covers:

- Group entry FQNs (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `semver.pkl` / `migrate.pkl`) and their exported field names
- The bundled entry `tasks/all.pkl` and the `kawaz.<group>.<field>` namespace it exposes
- Public task names (`vcs:commit`, `docs:check-translations`, `docs:check-translation-commit-lag`, `docs:check-translation-links`, `semver:compare`, `migrate:check-pkf-tasks-current`, etc.)

Internal implementation files (`vcs/iface.pkl`, `vcs/jj.pkl`, `vcs/git.pkl`, `vcs/auto.pkl`, `docs/translations.pkl`, `semver/check-bumped.pkl`, `migrate/check-current.pkl`, etc.) are **not** part of the public contract — they may move or be renamed in any minor/patch release. Always import via the flat group entries.

Pin to `pkf-tasks@2` for major-only floating, or to an exact version (e.g. `pkf-tasks@2.2.2`) for full reproducibility. The `migrate:check-*` gates keep the pin up to date regardless.

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
