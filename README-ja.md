# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。`kawaz/*` 系プロジェクト向けの jj/git auto-dispatch、ドキュメント整合性検査、lint、SemVer gate、upstream 追従検知を drop-in で提供。

## Modules

- [vcs](./tasks/vcs/) — jj/git auto-dispatch な VCS primitive + knowledge 集積場
- [docs](./tasks/docs/) — 翻訳ペア (`*-ja.md` / `*.md`) 整合性検査
- [lint](./tasks/lint/) — 言語横断 Pkl format + 孤児 module 検出
- [semver](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI 必須)
- [migrate](./tasks/migrate/) — pkf-tasks / pkfire の upstream 追従検知 + 自動修復 (`bump-semver` CLI 必須)

各モジュールディレクトリには task 詳細・パラメータ・利用例を載せた README がある。

## 使い方

### 集約 import (一括、`all.pkl` 経由)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterize して instance 化
}
```

### 個別 import (一部 group だけ使う場合)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/docs.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。

## SemVer 安定性 (1.x)

v1.0.0 以降は SemVer 2.0.0 に従う。1.x 系列の間は破壊的変更なしを保証する **public API contract** は次の範囲:

- group entry の FQN (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `lint.pkl` / `semver.pkl` / `migrate.pkl`) と export field 名
- 集約 entry の `tasks/all.pkl` と、そこから露出する `kawaz.<group>.<field>` namespace
- 公開 task 名 (`vcs:commit` / `docs:check-translations` / `lint:pkl` / `semver:compare` / `migrate:check-pkf-tasks-current` 等)

内部実装ファイル (`vcs/iface.pkl` / `vcs/jj.pkl` / `vcs/git.pkl` / `vcs/auto.pkl` / `docs/translations.pkl` / `lint/pkl.pkl` / `semver/check-bumped.pkl` / `migrate/check-current.pkl` 等) は public contract に **含まない** — minor / patch でいつでも改名・移動し得る。利用側は必ず flat な group entry 経由で import すること。

major 浮動が許容できるなら `pkf-tasks@1` で pin、完全再現したいなら exact pin (例: `pkf-tasks@1.0.0`)。どちらのケースでも `migrate:check-*` gate が pin の鮮度を保つ。

pkfire 0.6.0+ では CLI で **glob target** が使え、`<group>:<action>` 命名と相性が良い:

```bash
pkf run 'lint:*'            # 全 lint:* task を topological 順で
pkf run 'semver:*'          # 全 semver: gate / utility を一括
pkf list 'migrate:check-*'  # check-* 系の migration gate だけ確認
```

## 詳細

- [CHANGELOG](./CHANGELOG.md)
- [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) — 内部設計
- [docs/decisions/](./docs/decisions/) — Decision Record (DR)
- 実例: [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
