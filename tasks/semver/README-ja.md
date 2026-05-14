# `semver/` — SemVer gate + ad-hoc compare

> [English](./README.md) | 日本語

SemVer 比較を伴う gate と CLI ラッパを提供する group。`bump-semver` CLI が PATH 必須 (DR-0005)。pkf-tasks の責務は VCS dispatch + task 合成までで、SemVer 比較は bump-semver に任せる切り分け。fallback (Pkl で SemVer 比較を再実装する案) は採用しない。

主用途は 2 つ:

- **`semver:check-bumped`** — push 前 / release 前の version bump 漏れ gate。`object-amends` で per-instance parameterize できるよう **module 参照** として公開
- **`semver:compare`** — `bump-semver compare` の薄ラッパ (`acceptsArgs = true`)、ad-hoc CLI 利用専用

## Public entry と内部ファイル

- **Public entry**: `tasks/semver.pkl` — 利用者が `import` してよい唯一のファイル。export している field 名 (`checkBumped` module 参照 / `compare` task / `tasks`) は 1.x の public API contract に含まれる
- **内部実装** (外部から直接 import しない、minor release でも改名・移動し得る):
  - `check-bumped.pkl` — parameterize 可能な gate module。`(kawaz.semver.checkBumped) { ... }.check` で instance 化
  - `compare.pkl` — `semver:compare` task (ad-hoc CLI)

## 提供 task

### `semver:check-bumped`

- 便利度: ★★★
- 種別: **module 参照** (利用側で parameterize して instance 化する想定、`tasks` には含まない)
- 内部動作: 比較対象 ref から `triggerPaths` が変更されているかを `vcs.diffSummary` で検知し、変更ありなら `versionFiles` 全件について `bump-semver compare gt <file> vcs:<ref>:<file>` を判定。1 つでも未 bump なら fail
- パラメータ:
  - `compareRef: String` — 比較対象 VCS ref を表す plain な文字列 (e.g. `main@origin` / `origin/main` / `vcs:latest-tag(kawaz/pkf-tasks)` のような関数呼び出し)。default `"main@origin"` (push 前ガード、v3.0+)。旧 `compareRefCmd` (shell command substitution の中身を受ける形) は v3.0 で削除された (利用者が ref 文字列ではなく 10 行近いシェルスクリプトを書き始める運用ミスが多発したため)
  - `triggerPaths: List<String>` — 変更検知の対象 paths。default `List("src/")`
  - `versionFiles: List<String>` — bump 対象 (e.g. `VERSION` / `Cargo.toml` / `package.json`)。default `List("VERSION")`
  - `taskName: String` — task 名。複数 instance を作るときに別名を付ける。default `"semver:check-version-bumped"`
  - `needsFetchTags: Boolean` — true で `vcs.fetchTags` を deps に挟む。tag 系 ref (`git tag -l ...` / `vcs:latest-tag()`) を使うとき有効化。default `false`
- 設計判断: `bump-semver` 未インストール時は `ERROR: bump-semver not installed. SemVer comparison fallback is not implemented.` で停止 (DR-0005)。pkf-tasks 内に SemVer 比較ロジックを抱え込まない

### `semver:compare`

- 便利度: ★★
- 種別: task 直接公開 (`acceptsArgs = true` で `--` 以降の引数を bump-semver にパススルー)
- 内部動作: `bump-semver compare "$@" --no-hint` を実行
- 利用例:
  - `pkf run semver:compare -- gt VERSION 1.0.0`
  - `pkf run semver:compare -- lt Cargo.toml vcs:latest-tag():Cargo.toml`
  - `pkf run semver:compare -- eq package.json vcs:main@origin:package.json`
- `<INPUT>` 形式 (bump-semver から継承): `FILE` (basename 自動判定) / 生 semver / `-` (stdin) / `vcs:REV[:FILE]` / `vcs:latest-tag()`
- 設計判断: pkfire 仕様上 `deps` は `Listing<Task>` で引数渡し不可なため、本 task は ad-hoc CLI 利用専用。`check-bumped` のような複合 gate は本 task を deps として再利用できないので、内部で `bump-semver compare` を直接呼ぶ別実装になっている
- 注意: `cache = true` (pkfire の `$@` args が cache key に入る)。tag 移動などで結果が変わるのは利用者責任

## 利用例 (group 全体)

`semver:compare` だけは `tasks` 経由で一括登録、`check-bumped` は instance 化:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.10.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/all.pkl" as kawaz

// push 前ガード (main@origin と比較)
local checkPush = (kawaz.semver.checkBumped) {
  compareRef = "main@origin"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")
  taskName = "semver:check-version-bumped"
}.check

// release 前ガード (bump-semver の vcs: 関数で最新 semver tag と比較、tag fetch が必要)
local checkRelease = (kawaz.semver.checkBumped) {
  compareRef = "vcs:latest-tag()"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")
  taskName = "semver:check-against-latest-release"
  needsFetchTags = true
}.check

tasks {
  ...kawaz.semver.tasks   // semver:compare のみ登録
  checkPush                  // instance task を個別追加
  checkRelease
}
```

`semver:check-bumped` を `tasks` に含めないのは、利用側 project の version ファイルパスが分からないと意味のある instance を作れないため (default は `VERSION` だが、`Cargo.toml` 等の場合は parameterize 必須)。

## 関連

- [上位 README](../../README-ja.md)
- DR-0005 semver group 新設、bump-semver fallback 不採用
- DR-0007 group 構造規約 (flat `<group>.pkl` entry、内部 `<group>/...pkl` は public API ではない)
- DR-0019 bump-semver 側の `vcs:latest-tag(<arg>)` ref schema (`migrate:check-*` の最新 tag 取得で利用)
- 関連 module: vcs group の `diffSummary` を内部で使用
- 関連 CLI: [`kawaz/bump-semver`](https://github.com/kawaz/bump-semver) (`brew install kawaz/tap/bump-semver`)
