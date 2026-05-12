# `lint/` — 言語横断 lint + meta lint

> [English](./README.md) | 日本語

任意プロジェクトに共通する lint (Pkl format) と、pkf-tasks library 自身の整合性チェック (孤児 module 検出) を提供する。言語ごとの lint (gofmt / cargo fmt / rustfmt 等) は本 group に含めず、利用側で `lint:pkl` を自前 umbrella task の deps に並べる運用。

## Public entry と内部ファイル

- **Public entry**: `tasks/lint.pkl` — 利用者が `import` してよい唯一のファイル。export している field 名 (`pkl` / `allCoverage` / `allTasks`) は 1.x の public API contract に含まれる
- **内部実装** (外部から直接 import しない、minor release でも改名・移動し得る):
  - `pkl.pkl` — `lint:pkl` task (Pkl ファイル全体に `pkl format -w` を当てる)
  - `all-coverage.pkl` — `lint:all-coverage` task (library 開発者向け、孤児 module 検出)

## 提供 task

### `lint:pkl`

- 便利度: ★★★
- 内部動作: `pkl format -w .` を cwd 起点で再帰実行
- inputs: `**/*.pkl` / `**/PklProject` / `**/PklProject.deps.json` (pkfire の doublestar 構文)
- 利用例: 利用側の `lint` umbrella task の deps に組み込む

```pkl
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz
local lint = new Taskfile.Task {
  name = "lint"
  deps { goLint; kawaz.lint.pkl }
  cmd = "echo lint ok"
}
tasks { lint; goLint; kawaz.lint.pkl }
```

- 注意: pkl 0.31+ 前提 (pkfire の minPklVersion と同じ)。個別ファイルだけ整形したい場合は `(format) { cmd = "pkl format -w specific.pkl" }` で上書き

### `lint:all-coverage`

- 便利度: ★ (library 開発者向け、利用者は通常不要)
- 内部動作: `tasks/` 配下の `.pkl` のうち、他の `.pkl` から basename 言及されていない module を「孤児」として検出して fail。`find ... -exec grep -l ...` で参照を 1 件でも見つければ ok
- 除外対象: `PklProject*` / `Taskfile.pkl` / `iface.pkl` / `all.pkl`
- 設計判断: 0.0.10 で導入時は「all.pkl 群限定の参照」を要求していたが、`vcs/jj.pkl` のような iface 実装 (auto.pkl が `import` するだけで all.pkl には直接登録しない) が false-positive で落ちる問題があり、0.0.11 で「全 .pkl 範囲での参照」に緩和
- 設計判断: 検出のみで fail し、自動修復 (`fix:all-coverage`) は将来別 task として提供する想定。gate と action を分けるのは `semver:check-bumped` と `bump-version`、`migrate:check-*` と `migrate:update-*` と同じ流儀

## 利用例 (group 全体)

```pkl
tasks {
  ...kawaz.lint.allTasks   // lint:pkl + lint:all-coverage の両方登録
}
```

利用者プロジェクトでは通常 `kawaz.lint.pkl` だけを自前 umbrella の deps に挟めば十分。`lint:all-coverage` は pkf-tasks 自身の dogfood (CI gate) で使う。

## 関連

- [上位 README](../../README-ja.md)
- DR-0007 group 構造規約 (flat `<group>.pkl` entry、内部 `<group>/...pkl` は public API ではない)
