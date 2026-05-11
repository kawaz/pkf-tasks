# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。提供 module:

- **`vcs/auto.pkl`** — jj/git auto-dispatch な Task primitive (`commit` / `push` / `fetch` / `ensureClean`) と、他 Task の cmd 内で `\(vcs.diffSummary(...))` のように埋め込んで使う Pkl function helper (`diffSummary` / `readAtRef`)。
- **`docs/translations.pkl`** — 翻訳ペア (`*-ja.md` / `*.md`) の存在 / 相互リンク / commit timestamp 順序整合チェック。
- **`lint/pkl.pkl`** — 言語横断 `pkl format -w` task。
- **`semver/check-bumped.pkl`** — `compareRef` 以降に `triggerPaths` が変わったのに VERSION ファイルが bump されてなければ fail する parameterized gate。`bump-semver` CLI 必須。

## 使い方

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

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。0.0.x は不安定なので exact pin 推奨。

詳細: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) / [docs/decisions/](./docs/decisions/) / 実例 [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
