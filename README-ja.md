# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。提供 group:

- **`vcs/`** — jj/git auto-dispatch な Task primitive (`commit` / `push` / `fetch` / `ensureClean` / `fetch-tags`) と、他 Task の cmd 内で `\(vcs.diffSummary(...))` のように埋め込んで使う Pkl function helper (`diffSummary` / `readAtRef`)。jj/git 操作 knowledge の集積場 (DR-0006)。
- **`docs/translations.pkl`** — 翻訳ペア (`*-ja.md` / `*.md`) の存在 / 相互リンク / commit timestamp 順序整合チェック。
- **`lint/`** — `lint:pkl` (`pkl format -w`) と `lint:all-coverage` (パッケージ内孤児 module 検出)。
- **`semver/`** — parameterized な SemVer gate (`check-bumped` で push 時 / release 時の VERSION bump 強制) と `bump-semver compare` の薄いラッパ `compare` task。`bump-semver` CLI が必要。
- **`migrate/`** — `migrate:check-pkf-tasks-current` (gate、Taskfile.pkl の pkf-tasks import が最新 release より古いと fail) と `migrate:update-pkf-tasks` (action、import を最新 tag に sed 書き換え)。`push` の deps に `migrate:check` を挟むことで「気づき発火点」として機能。

## 使い方

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
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterize して instance 化
}
```

集約 endpoint `all.pkl` (v0.0.10+) で全 group に `kawaz` 1 namespace 経由でアクセス。個別 import (`import ".../vcs/auto.pkl" as vcs`) も従来どおり可。

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。0.0.x は不安定なので exact pin 推奨、`migrate:check-pkf-tasks-current` が pin の鮮度を保つ。

詳細: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) / [docs/decisions/](./docs/decisions/) / 実例 [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
