# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。`kawaz/*` 系プロジェクト向けの jj/git auto-dispatch、ドキュメント整合性検査、lint、SemVer gate、upstream 追従検知を drop-in で提供。

## Modules

- [`tasks/vcs/`](./tasks/vcs/) — jj/git auto-dispatch な VCS primitive + knowledge 集積場
- [`tasks/docs/`](./tasks/docs/) — 翻訳ペア (`*-ja.md` / `*.md`) 整合性検査
- [`tasks/lint/`](./tasks/lint/) — 言語横断 Pkl format + 孤児 module 検出
- [`tasks/semver/`](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI 必須)
- [`tasks/migrate/`](./tasks/migrate/) — pkf-tasks / pkfire の upstream 追従検知 + 自動修復 (`bump-semver` CLI 必須)

各モジュールディレクトリには task 詳細・パラメータ・利用例を載せた README がある。

## 使い方

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
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterize して instance 化
}
```

集約 endpoint `all.pkl` で全 group に `kawaz` 1 namespace 経由でアクセス。個別 import (`import ".../vcs/auto.pkl" as vcs`) も従来どおり可。

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。0.0.x は不安定なので exact pin 推奨、`migrate:check-*` gate が pin の鮮度を保つ。

pkfire 0.6.0+ では CLI で **glob target** が使え、`<group>:<action>` 命名と相性が良い:

```bash
pkf run 'lint:*'            # 全 lint:* task を topological 順で
pkf run 'semver:*'          # 全 semver: gate / utility を一括
pkf list 'migrate:check-*'  # check-* 系の migration gate だけ確認
```

詳細: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) / [docs/decisions/](./docs/decisions/) / 実例 [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
