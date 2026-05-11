# pkf-tasks 設計書

> [English](./DESIGN.md) | 日本語

`kawaz/pkf-tasks` の内部構造と設計判断の集約。利用者向けの使い方は [README-ja.md](../README-ja.md) を、release ごとの変更は [CHANGELOG.md](../CHANGELOG.md) を参照。

## モジュール構成

```
tasks/
├── PklProject               # Pkl package metadata (packageUri / dependencies / version)
├── PklProject.deps.json     # 解決済 dep lockfile (SHA256 pin)
├── vcs/
│   ├── iface.pkl            # abstract module (commit / push / fetch / ensureClean)
│   ├── jj.pkl               # iface を extends した jj 実装
│   ├── git.pkl              # iface を extends した git 実装
│   └── auto.pkl             # entry。jj / git の cmd を bash if-else で連結
└── docs/
    └── translations.pkl     # README{,-ja}.md 等のペア検証 (1 Task で bash for)
```

## VCS 抽象 — abstract module + extends + 実行時切替

`tasks/vcs/iface.pkl` で `abstract module` として interface を宣言、jj.pkl / git.pkl が `extends "iface.pkl"` で実装。`amends` ではない (Pkl の abstract module は `extends` でしか実装できない、詳細は [DR-0001](./decisions/DR-0001-abstract-module-extends.md))。

`auto.pkl` は jj 版と git 版の両モジュールを `import` し、各 task の cmd を bash の if-else で連結する `autoCmd(jjCmd, gitCmd)` で組み立てる。Pkl 評価時に jj / git を選んで切り替えるのではなく、生成される cmd 自身が `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi` を含む。Pkl が pure evaluation でファイルシステム状態を見られないことへの対応 (詳細は [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md))。

検出順は `.jj` 優先 → `.git`。`jj root` / `git rev-parse --git-dir` を使う理由は、cwd 直下に `.jj` / `.git` が無くても親方向に upward search してくれるため (`jj workspace add` で作ったサブディレクトリ問題への対応)。

`(jj.commit) { description = ...; cmd = autoCmd(...) }` という Pkl の object-amends 構文で「jj 版を base に description と cmd だけ上書き」し、`params` / `env` / `cache` / `shell` 等は jj 版から継承する。

## 翻訳ペア検証 — 1 Task で bash for ループ

`tasks/docs/translations.pkl` は `README{,-ja}.md` / `docs/DESIGN{,-ja}.md` / `docs/MANUAL{,-ja}.md` 等の対をまとめて検証する。Pkl レベルでは 1 task 定義のみ、内部の bash の `for pair in ...` ループで各ペアを処理。

`Listing<Task>` 展開せずに 1 Task で済ませる理由: 翻訳チェックは inputs cache の独立粒度が要らない (どれか 1 ペアが変わっても全ペア再チェックして良い軽い処理) ため、Pkl 側でサブタスクに分割するメリットがない。pkfire 流儀の「cache 独立性を必要とする処理だけ Task を分割する」原則に沿う。

検証内容:

1. `<pair>-ja.md` が無ければスキップ (任意ペア対応)
2. 同じディレクトリに `<pair>.md` (英訳) が存在
3. `<pair>-ja.md` の冒頭 5 行に `> [English](./<base>.md) | 日本語`
4. `<pair>.md` の冒頭 5 行に `> English | [日本語](./<base>-ja.md)`
5. commit timestamp が `ja_ts <= en_ts` (英訳が遅れていないこと)

`set -euo pipefail` + `failed=0` accumulator + `exit $failed` で **全ペアを検査した上で 1 つでも fail なら exit 1**。`set -e` のみだと最初の fail で残りペアが見えなくなるため、両者の合わせ技で堅牢化 ([CHANGELOG 0.0.4](../CHANGELOG.md#004--2026-05-11))。

## 配布 — Pkl package + GitHub Release

`tasks/PklProject` で package metadata を宣言:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "0.0.5"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

tag 命名は `pkf-tasks@<version>` (mizchi/pkfire の流儀踏襲、詳細は [DR-0003](./decisions/DR-0003-tag-naming.md))。`pkl project package` で zip + metadata json + SHA256 を生成し、GitHub Release に asset としてアップロード。`pkg.pkl-lang.org` proxy 経由で利用側の `pkl eval` / `pkf list` が取得する。

利用側の Pkl cache は `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/` で、SHA256 検証付きで保存される。一度取得すれば Pkl サーバが落ちていてもオフライン評価可能。

## CI / リリース自動化

`.github/workflows/ci.yml`: main へ push / PR 時に pkfire の composite action (`mizchi/pkfire@pkfire@0.4.0`) で pkl + pkf を準備、各 module を `pkl eval` で smoke-test、`pkl project package` の生成可否を確認、自リポの翻訳ペアの整合性 (self dogfooding) を確認。

`.github/workflows/release.yml`: `pkf-tasks@*` tag が push されたら自動で `pkl project package` を実行し、生成された 4 asset を GitHub Release にアップロード。tag と PklProject の version が一致しない場合は fail させて誤 publish を防ぐ。

## 関連ドキュメント

- 設計判断記録: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- 変更履歴: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* 共通の docs 構造規約: `~/.claude/rules/docs-structure.md`
