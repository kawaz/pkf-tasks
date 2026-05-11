# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Modules provided:

- **`vcs/auto.pkl`** — jj/git auto-dispatched task primitives (`commit` / `push` / `fetch` / `ensureClean`) plus Pkl function helpers (`diffSummary` / `readAtRef`) for embedding inside other Task `cmd`s.
- **`docs/translations.pkl`** — translation-pair integrity check (paired `*-ja.md` / `*.md` documents — existence, mutual link, commit-timestamp order).
- **`lint/pkl.pkl`** — language-agnostic `pkl format -w` task.
- **`semver/check-bumped.pkl`** — parameterised gate task that fails if `triggerPaths` changed since `compareRef` without a corresponding VERSION bump. Requires `bump-semver` CLI on PATH.

## Usage

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.4.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.7#/vcs/auto.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.7#/docs/translations.pkl" as docs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.7#/lint/pkl.pkl" as lintPkl

tasks {
  ...vcs.allTasks
  docs.checkTranslations
  ...lintPkl.allTasks
}
```

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases). Pin to an exact version while in the 0.0.x phase.

More: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN.md](./docs/DESIGN.md) / [docs/decisions/](./docs/decisions/) / example consumer [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
