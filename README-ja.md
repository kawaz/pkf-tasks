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
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.10.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.tasks
  ...kawaz.docs.tasks
  ...kawaz.migrate.tasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRef = "main@origin"; ... }.check  // parameterize して instance 化
}
```

### 個別 import (一部 group だけ使う場合)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.10.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/vcs.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/docs.pkl" as docs

tasks {
  ...vcs.tasks
  docs.checkTranslations
}
```

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。

### `docs:check-translations` の使い方 (v2.1.0+)

`acceptsArgs = true` で正本リスト / glob を CLI で渡せる。サブチェック (`commit-lag` / `links`) はそれぞれ独立に同じ引数を受け取る。umbrella の `docs:check-translations` は CLI args を deps に伝播しない (pkfire 0.6.0 の orchestrator 制約)。CLI で glob を渡したい時はサブチェックを直接呼ぶ:

```bash
# auto-discover (cwd 配下の *-ja.md を全て正本扱い、.jj/ .git/ .out/ node_modules/ は除外)
pkf run docs:check-translations

# 明示リスト (kawaz の慣習通り)
pkf run docs:check-translation-commit-lag -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# 雑に全部 (** は bash 4+、bash 3.2 では 1 階層のみ — brace expansion {,*/,*/*/}*-ja.md で代用可)
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'
pkf run docs:check-translation-links -- '**/*-ja.md'
```

#### 近所探索 (正本 → 翻訳先)

各正本に対して、同 basename のファイルを翻訳先として近所探索 (正本自身は除外):

| 正本パターン | 翻訳先 | 用途 |
|---|---|---|
| `<base>-ja.md` (kawaz 慣習) | `<base>.md` 単体 (1:1、en) | ja 原本プロジェクト |
| `<base>.md` (en 原本など) | `<base>-??.md` / `<base>-???.md` (2-3 文字 language code: `-ja.md`, `-zh.md`, `-jpn.md`, etc.) | en 原本 / 多言語 |

`-ja.md` 正本は **1:1 探索** (en のみ) に固定して、`data-layout-ja.md` と `data-layout-history-ja.md` が同じ翻訳ペアとして誤検知される問題を回避。`*.md` 正本は同様の理由で短い language-code suffix に限定。

相互リンク検査は ja ↔ en の 2 言語規約 (1 正本 ↔ 1 翻訳先) のみ。翻訳先が 0 個 (未対訳) or 2+ 個 (多言語) の場合は skip ログ + pass — リンク規約はそれらのケースには拡張されていない (将来課題、`docs/issue/2026-05-12-link-pattern-injection.md` 参照)。

## SemVer 安定性 (2.x)

v1.0.0 以降は SemVer 2.0.0 に従う。2.x 系列の間は破壊的変更なしを保証する **public API contract** は次の範囲:

- group entry の FQN (`tasks/<group>.pkl`: `vcs.pkl` / `docs.pkl` / `semver.pkl` / `migrate.pkl`) と export field 名
- 集約 entry の `tasks/all.pkl` と、そこから露出する `kawaz.<group>.<field>` namespace
- 公開 task 名 (`vcs:commit` / `docs:check-translations` / `docs:check-translation-commit-lag` / `docs:check-translation-links` / `semver:compare` / `migrate:check-pkf-tasks-current` 等)

内部実装ファイル (`vcs/iface.pkl` / `vcs/jj.pkl` / `vcs/git.pkl` / `vcs/auto.pkl` / `docs/translations.pkl` / `semver/check-bumped.pkl` / `migrate/check-current.pkl` 等) は public contract に **含まない** — minor / patch でいつでも改名・移動し得る。利用側は必ず flat な group entry 経由で import すること。

major 浮動が許容できるなら `pkf-tasks@3` で pin、完全再現したいなら exact pin (例: `pkf-tasks@3.0.0`)。どちらのケースでも `migrate:check-*` gate が pin の鮮度を保つ。

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
