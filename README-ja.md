# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。jj/git 抽象 (`vcs/auto.pkl`) と翻訳ペアチェック (`docs/translations.pkl`) を提供。

## 使い方

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.4.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.7#/vcs/auto.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.7#/docs/translations.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。0.0.x は不安定なので exact pin 推奨。

詳細: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) / [docs/decisions/](./docs/decisions/) / 実例 [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
