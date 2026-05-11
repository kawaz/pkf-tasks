# pkf-tasks

> English | [日本語](./README-ja.md)

Shared task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire). Provides `vcs/auto.pkl` (jj/git abstraction) and `docs/translations.pkl` (translation-pair integrity check).

## Usage

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.4.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.5#/vcs/auto.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.5#/docs/translations.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

Inspect with `pkf list` / `pkf list -v` / `pkf graph --format mermaid`. Latest versions live in [Releases](https://github.com/kawaz/pkf-tasks/releases). Pin to an exact version while in the 0.0.x phase.

More: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN.md](./docs/DESIGN.md) / [docs/decisions/](./docs/decisions/) / example consumer [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
