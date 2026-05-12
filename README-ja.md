# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。`kawaz/*` 系プロジェクト向けの jj/git auto-dispatch、ドキュメント整合性検査、SemVer gate、upstream 追従検知を drop-in で提供。

> kawaz 個人のワークフロー用に運用している共通タスク置き場であり、公共のリファレンス実装ではない (fork / 参考利用は歓迎)。命名や粒度は kawaz/* 各リポを揃えるのを優先する。

## Modules

- [vcs](./tasks/vcs/) — jj/git auto-dispatch な VCS primitive + knowledge 集積場
- [docs](./tasks/docs/) — 翻訳ペア (`*-ja.md` / `*.md`) 整合性検査 (v2.1.0+ で commit-lag / links に分割 + acceptsArgs)
- [semver](./tasks/semver/) — SemVer bump gate + ad-hoc compare (`bump-semver` CLI 必須)
- [migrate](./tasks/migrate/) — pkf-tasks / pkfire の upstream 追従検知 + 自動修復 (`bump-semver` CLI 必須)

各モジュールディレクトリには task 詳細・パラメータ・利用例を載せた README がある。

## 使い方

### 集約 import (一括、`all.pkl` 経由)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterize して instance 化
}
```

### 個別 import (一部 group だけ使う場合)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.1.0#/docs.pkl" as docs

tasks {
  ...vcs.allTasks
  docs.checkTranslations
}
```

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。

### `docs:check-translations` の使い方 (v2.1.0+)

`acceptsArgs = true` で正本リスト / glob を CLI で渡せる。`-ja.md` 形式の正本に対しては近所の `*.md` を翻訳先として近所探索する:

```bash
# auto-discover (cwd 配下の *-ja.md を全て正本扱い)
pkf run docs:check-translations

# 明示リスト (kawaz の慣習通り)
pkf run docs:check-translations -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# 雑に全部
pkf run docs:check-translations -- '**/*-ja.md'

# 個別 check (sub task)
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'   # commit-lag のみ (links は skip)
pkf run docs:check-translation-links -- README-ja.md         # links のみ (commit-lag は skip)
```

正本が `*.md` (en 原本など) の場合は `*[_.-]*.md` (`-ja.md` / `-zh.md` 等) を近所探索する形で多言語にも対応。相互リンクの検査は 2 言語ペアのみ (3+ 翻訳先がある正本は skip ログ + pass)。

## SemVer 安定性 (2.x)

v1.0.0 以降は SemVer 2.0.0 に従う。2.x 系列の間は破壊的変更なしを保証する **public API contract** は次の範囲:

- group entry の FQN (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `semver.pkl` / `migrate.pkl`) と export field 名
- 集約 entry の `tasks/all.pkl` と、そこから露出する `kawaz.<group>.<field>` namespace
- 公開 task 名 (`vcs:commit` / `docs:check-translations` / `docs:check-translation-commit-lag` / `docs:check-translation-links` / `semver:compare` / `migrate:check-pkf-tasks-current` 等)

内部実装ファイル (`vcs/iface.pkl` / `vcs/jj.pkl` / `vcs/git.pkl` / `vcs/auto.pkl` / `docs/translations.pkl` / `semver/check-bumped.pkl` / `migrate/check-current.pkl` 等) は public contract に **含まない** — minor / patch でいつでも改名・移動し得る。利用側は必ず flat な group entry 経由で import すること。

major 浮動が許容できるなら `pkf-tasks@2` で pin、完全再現したいなら exact pin (例: `pkf-tasks@2.1.0`)。どちらのケースでも `migrate:check-*` gate が pin の鮮度を保つ。

pkfire 0.6.0+ では CLI で **glob target** が使え、`<group>:<action>` 命名と相性が良い:

```bash
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
