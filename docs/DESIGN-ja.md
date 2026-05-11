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
│   └── auto.pkl             # entry。Task export + Pkl function helper (diffSummary / readAtRef)
├── docs/
│   └── translations.pkl     # README{,-ja}.md 等のペア検証 (1 Task で bash for)
├── lint/
│   └── pkl.pkl              # 言語横断 `pkl format -w` task
└── semver/
    └── check-bumped.pkl     # parameterized SemVer bump gate (bump-semver 必須)
```

モジュールの内部 FQN は `com.github.kawaz.pkfTasks.*` を使う。利用側は `import ... as <alias>` で alias import するため FQN の変更は影響しないが、reverse-domain notation としては `kawaz.com` を所有していないので `com.github.kawaz` に揃えるのが正確 (v0.0.6 で修正、利用者非破壊)。

## VCS 抽象 — abstract module + extends + 実行時切替

`tasks/vcs/iface.pkl` で `abstract module` として interface を宣言、jj.pkl / git.pkl が `extends "iface.pkl"` で実装。`amends` ではない (Pkl の abstract module は `extends` でしか実装できない、詳細は [DR-0001](./decisions/DR-0001-abstract-module-extends.md))。

`auto.pkl` は jj 版と git 版の両モジュールを `import` し、各 task の cmd を bash の if-else で連結する `autoCmd(jjCmd, gitCmd)` で組み立てる。Pkl 評価時に jj / git を選んで切り替えるのではなく、生成される cmd 自身が `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi` を含む。Pkl が pure evaluation でファイルシステム状態を見られないことへの対応 (詳細は [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md))。

検出順は `.jj` 優先 → `.git`。`jj root` / `git rev-parse --git-dir` を使う理由は、cwd 直下に `.jj` / `.git` が無くても親方向に upward search してくれるため (`jj workspace add` で作ったサブディレクトリ問題への対応)。

`(jj.commit) { description = ...; cmd = autoCmd(...) }` という Pkl の object-amends 構文で「jj 版を base に description と cmd だけ上書き」し、`params` / `env` / `cache` / `shell` 等は jj 版から継承する。

### Pkl function helper の二層 export

`vcs/auto.pkl` は **Task export** (`commit` / `push` / `fetch` / `ensureClean`) に加えて、**Pkl function** を同じ module から export する:

- `diffSummary(ref: String, paths: List<String>): String` — 任意の ref と paths を比較し変更ファイル一覧を返す bash command substitution (`$(...)` の中身)
- `readAtRef(ref: String, path: String): String` — 任意の ref におけるファイル内容を返す bash command substitution

これらは値そのものではなく **bash 断片** を返す。利用側は他 Task の cmd 内に `\(vcs.diffSummary("$ref", List("src/")))` のように Pkl 文字列補間で埋め込み、実行時に bash の if-else で jj/git dispatch される (auto-detect)。

helpers を `vcs/helpers.pkl` のような別 module に切り出さなかった理由: jj/git dispatch ロジックが auto.pkl と二重になる。auto.pkl を「auto-detect なら何でもここに集約」のシングルゲートウェイにする方が利用側の `import ... as vcs` 1 つで Task と helper の両方にアクセスできる。詳細は [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md)。

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

## 言語横断 Lint — `lint/pkl.pkl`

`tasks/lint/pkl.pkl` は `pkl format -w` を `**/*.pkl` + `PklProject` + `PklProject.deps.json` の inputs に対して再帰適用する Task (`lint:pkl`) を提供する。Pkl ファイルの整形は kawaz/* リポ間で共通の関心事なので、言語固有 lint (`gofmt` / `cargo fmt` 等) からは独立させて配布する。

利用側で言語固有 lint と組み合わせて umbrella `lint` task を構成する想定:

```pkl
local lint: Task = new {
  name = "lint"; cache = false
  deps { goLint; pklLint.format }
  cmd = "echo lint ok"
}
```

`pkl format` の動作差を抑えるため pkfire と同じ `minPklVersion = "0.31.0"` を前提にする。

## SemVer ゲート — `semver/check-bumped.pkl`

「比較対象 ref 以降に `triggerPaths` が変わったのに VERSION ファイルが bump されていなければ fail」を検査するゲート Task。push 前 (`compareRef = main@origin`) や CI release 前 (`compareRef = 直近の v* tag`) のガードとして使う。

パラメータ化されており、利用側で複数インスタンスを切れる:

- `compareRefCmd: String` — 比較対象 ref を返す bash command substitution の中身
- `triggerPaths: List<String>` — 変更検知対象 (default `src/`)
- `versionFiles: List<String>` — bump 対象 VERSION ファイル一覧 (複数可)
- `taskName: String` — task 名 (利用側で `semver:check-version-bumped` / `semver:check-against-latest-release` のように別名を付ける)

`(semverCheck) { ... }.check` の object-amends で利用側が 2 task に展開する流儀。実装は `vcs.diffSummary` で trigger paths の差分を取り、空でなければ `bump-semver compare gt VERSION vcs:"$compare_ref":VERSION` で SemVer 比較する。

### bump-semver 必須 — SemVer 比較 fallback は意図的に未実装

`bump-semver` CLI が無ければ `not implemented: install bump-semver` メッセージで停止する。pure-Pkl / bash の fallback (`sort -V` 等) は実装しない:

- SemVer は prerelease (`-rc.1`) / build metadata (`+sha.abc`) の比較ルールが複雑で、`sort -V` は `0.14.0-rc.1` と `0.14.0` の順序が直感に反する
- `bump-semver` は semver.org 準拠で実装されており、責務をそちらに集約する方が DRY
- pkf-tasks の責務は VCS dispatch まで、SemVer 比較は bump-semver に任せる

利用側は `brew install kawaz/tap/bump-semver` 等で導入。CI 環境でも同様。`semver/*` は `vcs/*` に依存する一方向 layering で、逆依存はない。詳細は [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md)。

## 配布 — Pkl package + GitHub Release

`tasks/PklProject` で package metadata を宣言:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "0.0.7"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

tag 命名は `pkf-tasks@<version>` (mizchi/pkfire の流儀踏襲、詳細は [DR-0003](./decisions/DR-0003-tag-naming.md))。`pkl project package` で zip + metadata json + SHA256 を生成し、GitHub Release に asset としてアップロード。`pkg.pkl-lang.org` proxy 経由で利用側の `pkl eval` / `pkf list` が取得する。

利用側の Pkl cache は `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/` で、SHA256 検証付きで保存される。一度取得すれば Pkl サーバが落ちていてもオフライン評価可能。

## CI / リリース自動化

`.github/workflows/ci.yml`: main へ push / PR 時に pkfire の composite action (`mizchi/pkfire@<commit SHA>`) で pkl + pkf を準備、各 module を `pkl eval` で smoke-test、`pkl project package` の生成可否を確認、自リポの翻訳ペアの整合性 (self dogfooding) を確認。

`uses:` を tag (`mizchi/pkfire@pkfire@0.4.0`) ではなく **commit SHA で pin** する。pkfire の release tag に `@` が含まれる (`pkfire@0.4.0`) ことが GitHub Actions の workflow parser を壊す (workflow file レベルで fail し log すら取れない) ため。SHA pin は副次効果として supply chain security 観点でも推奨される。詳細は [DR-0004](./decisions/DR-0004-pkfire-action-sha-pin.md)。

`.github/workflows/release.yml`: `pkf-tasks@*` tag が push されたら自動で `pkl project package` を実行し、生成された 4 asset を GitHub Release にアップロード。tag と PklProject の version が一致しない場合は fail させて誤 publish を防ぐ。

## 関連ドキュメント

- 設計判断記録: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- 変更履歴: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* 共通の docs 構造規約: `~/.claude/rules/docs-structure.md`
