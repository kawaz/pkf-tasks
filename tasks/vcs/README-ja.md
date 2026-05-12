# `vcs/` — jj/git auto-dispatch な VCS primitive 群

> [English](./README.md) | 日本語

jj / git どちらの管理リポでも透過的に動く VCS task と、`bump-semver` 等の他ツールに任せきれない VCS knowledge を集める場 (DR-0006)。abstract module + extends で interface を切り出し (DR-0001)、`auto.pkl` が**実行時** (Task の cmd 内) に `jj root` / `git rev-parse --git-dir` で分岐する (DR-0002)。

Pkl は pure 評価で FS state を見られないため、評価時に jj/git を選んで module を切り替えるアプローチは不可。代わりに jj.pkl / git.pkl 双方から cmd を取り出し、bash の if-else で連結した cmd を提供する。`.jj` を `.git` より優先 (kawaz の git-bare + jj-workspace 構成は両方並存するが jj が正)。

## Public entry と内部ファイル

- **Public entry**: `tasks/vcs.pkl` — 利用者が `import` してよい唯一のファイル。export している field 名は 1.x の public API contract に含まれる
- **内部実装** (外部から直接 import しない、minor release でも改名・移動し得る):
  - `iface.pkl` — abstract module。`commit` / `push` / `fetch` / `ensureClean` / `fetchTags` の interface 定義 + `allTasks: Listing<Task>` 集約プロパティ
  - `jj.pkl` — jj 実装 (`extends "iface.pkl"`)
  - `git.pkl` — git 実装 (`extends "iface.pkl"`)
  - `auto.pkl` — 実行時 dispatch の実体。jj.pkl の各 Task 構造 (params / env / cache) を base に、cmd と description を auto-detect 版で上書き。`diffSummary` / `readAtRef` の Pkl helper 関数も提供。`tasks/vcs.pkl` が re-export する

## 提供 task

### `vcs:commit`

- 便利度: ★★★
- 引数: `message` (必須、commit message body)
- 内部動作: jj は `jj describe -m "$MESSAGE"` → `jj new`、git は `git commit -m "$MESSAGE"`
- 利用例: `pkf run vcs:commit -- --message='Release v0.1.0'`

### `vcs:push`

- 便利度: ★★★
- 内部動作: jj は `jj bookmark set main -r @-` → `jj git push --bookmark main`、git は `git push origin main`
- 利用例: 利用側の `push` umbrella task の最終段に `(vcs.push) { deps { ... } }` で extend し、前段に lint/test/check-bumped 等の gate を組む
- 注意: SSH agent socket はホスト inherit (pkfire の `inheritEnv = true` デフォルト) に任せる。`env` で明示すると CI 等で `read("env:...")` が resource not found になり eval fail する

### `vcs:fetch`

- 便利度: ★★
- 内部動作: jj は `jj git fetch`、git は `git fetch`

### `vcs:ensure-clean`

- 便利度: ★★★
- 内部動作: jj は `[ "$(jj log -r @ --no-graph -T 'empty')" = "true" ]`、git は `[ -z "$(git status --porcelain)" ]`
- 利用例: `push` の gate に組み込む

### `vcs:fetch-tags`

- 便利度: ★★
- 内部動作: jj は `jj git fetch || true` → `jj git import || true`、git は `git fetch --tags origin`
- 設計判断 (DR-0006 knowledge storage): jj は `remote.<name>.tagOpt` 未設定だと `jj git fetch` で tag が来ない。`jj git import` を併用して bare git 側で fetched された tag を jj 側にも反映する。両者 `|| true` で best-effort (ネットワーク不通でも後続が走れる)
- 利用例: `semver:check-bumped` の `needsFetchTags = true` 時に自動 deps、`vcs:latest-tag()` 系の前段

## 提供 Pkl helper 関数 (内部用)

cmd 文字列に `$(...)` として展開する command-substitution を返す helper。利用側は他 Task の cmd 内に文字列補間で埋め込む。

### `vcs.diffSummary(ref: String, paths: List<String>): String`

任意の ref と paths を比較し、変更があったファイル名一覧を返す bash command substitution。jj は `jj diff -r "<ref>..@" --summary -- <paths>`、git は `git diff --name-only "<ref>" -- <paths>` で実装。

利用例 (cmd 内):

```pkl
cmd = """
  diff_out=\(vcs.diffSummary("main@origin", List("src/")))
  [ -z "$diff_out" ] && exit 0
  # ... bump check etc.
"""
```

### `vcs.readAtRef(ref: String, path: String): String`

任意 ref におけるファイル内容を返す bash command substitution。jj は `jj file show -r <ref> -- <path>`、git は `git show <ref>:<path>`。

## 利用例 (group 全体)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks   // commit/push/fetch/ensure-clean/fetch-tags 一括登録
}
```

個別 import (vcs group のみ):

```pkl
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/vcs.pkl" as vcs
tasks { vcs.push; vcs.ensureClean }
```

## 関連

- [上位 README](../../README-ja.md)
- DR-0001 abstract module + extends パターン
- DR-0002 runtime VCS dispatch
- DR-0005 helper 関数 (`diffSummary` / `readAtRef`) を vcs group に置く判断
- DR-0006 vcs as knowledge storage
- DR-0007 group 構造規約 (flat `<group>.pkl` entry、内部 `<group>/...pkl` は public API ではない)
