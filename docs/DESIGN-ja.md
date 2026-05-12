# pkf-tasks 設計書

> [English](./DESIGN.md) | 日本語

`kawaz/pkf-tasks` の内部構造と設計判断の集約。利用者向けの使い方は [README-ja.md](../README-ja.md) を、release ごとの変更は [CHANGELOG.md](../CHANGELOG.md) を参照。本書は v2.x 時点の library 実装を説明する。

## モジュール構成

```
tasks/
├── PklProject               # Pkl package metadata (packageUri / dependencies / version)
├── PklProject.deps.json     # 解決済 dep lockfile (SHA256 pin)
├── all.pkl                  # root 集約 — `import .../all.pkl as kawaz` の単一窓口
├── vcs.pkl                  # vcs group entry (旧 vcs/all.pkl、v1.0.0 で flat 化)
├── vcs/
│   ├── iface.pkl            # abstract module (commit / push / fetch / ensureClean / fetchTags)
│   ├── jj.pkl               # iface を extends した jj 実装
│   ├── git.pkl              # iface を extends した git 実装
│   └── auto.pkl             # entry — Task export + Pkl function helper (diffSummary / readAtRef)
├── docs.pkl                 # docs group entry
├── docs/
│   └── translations.pkl     # README{,-ja}.md 等のペア検証 (1 Task で bash for)
├── semver.pkl               # semver group entry
├── semver/
│   ├── check-bumped.pkl     # parameterized SemVer bump gate (bump-semver 必須)
│   └── compare.pkl          # `semver:compare` — bump-semver compare の薄いラッパ (acceptsArgs)
├── migrate.pkl              # migrate group entry
└── migrate/
    ├── check-current.pkl    # `migrate:check-pkf-tasks-current` — pkf-tasks import 鮮度 gate
    └── update-self.pkl      # `migrate:update-pkf-tasks` — Taskfile.pkl の import を最新書換
```

v1.0.0 で sub `<group>/all.pkl` 集約は `tasks/<group>.pkl` に flat 化された (CHANGELOG 参照)。`lint` group は **v2.0.0 で library export から削除** された (`pkl format -w` と孤児 module 検出は pkf-tasks 自身の `Taskfile.pkl` 内 task に inline 化、library としては配布しない。詳細は [DR-0008](./decisions/DR-0008-lint-group-removed-from-library.md))。

モジュールの内部 FQN は `com.github.kawaz.pkfTasks.*` を使う。利用側は `import ... as <alias>` で alias import するため FQN の変更は影響しないが、reverse-domain notation としては `kawaz.com` を所有していないので `com.github.kawaz` に揃えるのが正確 (v0.0.6 で修正、利用者非破壊)。

## 集約 import — `all.pkl` の二層構造

利用側で `import "package://...pkf-tasks@<version>#/all.pkl" as kawaz` 1 行で全 group にアクセスできるようにする root 集約 (v0.0.10 で導入)。kawaz/* の各リポは pkf-tasks の全機能を使う前提なので、個別 import (`vcs/auto.pkl as vcs`, `docs/translations.pkl as docs`, ...) より集約 import の方が DRY。

```pkl
import "package://...pkf-tasks@2.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks                                       // commit / push / fetch / ensureClean / fetchTags
  ...kawaz.docs.allTasks                                      // checkTranslations
  kawaz.semver.compare                                        // ad-hoc CLI 用 (acceptsArgs)
  (kawaz.semver.checkBumped) { compareRefCmd = "..." }.check  // parameterize は module 参照
  ...kawaz.migrate.allTasks                                   // check + update
}
```

二層構造: **root `tasks/all.pkl`** が各 group の `tasks/<group>.pkl` を `import` して名前空間として再 export し、各 group entry がその group の module 群を一括公開する (v1.0.0 で sub `<group>/all.pkl` から `tasks/<group>.pkl` flat 化、v2.0.0 で `lint` group 削除 — [DR-0007](./decisions/DR-0007-group-structure-conventions.md) / [DR-0008](./decisions/DR-0008-lint-group-removed-from-library.md))。

### group entry での「task 直接公開」vs「module 参照公開」

各 group entry (`tasks/<group>.pkl`) は 2 通りの公開スタイルを使い分ける:

- **task 直接公開** — `kawaz.semver.compare` (acceptsArgs Task) のように `tasks { kawaz.semver.compare }` でそのまま登録できる短縮形。
- **module 参照公開** — `kawaz.semver.checkBumped` は **module** として公開し、利用側で `(kawaz.semver.checkBumped) { compareRefCmd = ...; versionFiles = ... }.check` のように parameterize して instance task を作る前提。

加えて各 group entry は `allTasks: Listing<Task>` を持ち、`tasks { ...kawaz.vcs.allTasks }` のような spread 一括登録に対応する (v0.0.11 で追加)。parameterize 前提の module 参照は `allTasks` から除外し、利用者が明示インスタンス化する責務にする。

## VCS 抽象 — abstract module + extends + 実行時切替

`tasks/vcs/iface.pkl` で `abstract module` として interface を宣言、jj.pkl / git.pkl が `extends "iface.pkl"` で実装。`amends` ではない (Pkl の abstract module は `extends` でしか実装できない、詳細は [DR-0001](./decisions/DR-0001-abstract-module-extends.md))。

`auto.pkl` は jj 版と git 版の両モジュールを `import` し、各 task の cmd を bash の if-else で連結する `autoCmd(jjCmd, gitCmd)` で組み立てる。Pkl 評価時に jj / git を選んで切り替えるのではなく、生成される cmd 自身が `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi` を含む。Pkl が pure evaluation でファイルシステム状態を見られないことへの対応 (詳細は [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md))。

検出順は `.jj` 優先 → `.git`。`jj root` / `git rev-parse --git-dir` を使う理由は、cwd 直下に `.jj` / `.git` が無くても親方向に upward search してくれるため (`jj workspace add` で作ったサブディレクトリ問題への対応)。

`(jj.commit) { description = ...; cmd = autoCmd(...) }` という Pkl の object-amends 構文で「jj 版を base に description と cmd だけ上書き」し、`params` / `env` / `cache` / `shell` 等は jj 版から継承する。

### Pkl function helper の二層 export

`vcs/auto.pkl` は **Task export** (`commit` / `push` / `fetch` / `ensureClean` / `fetchTags`) に加えて、**Pkl function** を同じ module から export する:

- `diffSummary(ref: String, paths: List<String>): String` — 任意の ref と paths を比較し変更ファイル一覧を返す bash command substitution (`$(...)` の中身)
- `readAtRef(ref: String, path: String): String` — 任意の ref におけるファイル内容を返す bash command substitution

これらは値そのものではなく **bash 断片** を返す。利用側は他 Task の cmd 内に `\(vcs.diffSummary("$ref", List("src/")))` のように Pkl 文字列補間で埋め込み、実行時に bash の if-else で jj/git dispatch される (auto-detect)。

helpers を `vcs/helpers.pkl` のような別 module に切り出さなかった理由: jj/git dispatch ロジックが auto.pkl と二重になる。auto.pkl を「auto-detect なら何でもここに集約」のシングルゲートウェイにする方が利用側の `import ... as vcs` 1 つで Task と helper の両方にアクセスできる。詳細は [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md)。

### `vcs/*` を VCS knowledge 集積場として位置付ける

higher-level なレシピ (`semver/check-bumped` 等) を書くたびに「jj だとこうやる / git だとこう」の bash 分岐をレシピ内に埋め込むと、同じ knowledge が複数レシピで重複して肥大化する。v0.0.9 で **`vcs/{iface,jj,git,auto}.pkl` を VCS dispatch + jj/git knowledge 集積場として明示的に位置付け** ([DR-0006](./decisions/DR-0006-vcs-as-knowledge-storage.md))、jj/git で挙動差異がある操作はここに集約する方針を確立。

吸収形態は 2 通り:

- **Task として追加** — 利用側 task の deps に挟みたい (=順序 + cache key で扱いたい) 場合
- **Pkl function として追加** — 利用側 task の cmd 内で `$(...)` 展開して使いたい (=文字列断片として組み立てる) 場合 (DR-0005)

具体例として v0.0.9 で追加した **`vcs:fetch-tags`** (`abstract fetchTags: Task` を iface に追加):

- jj: `jj git fetch || true; jj git import || true` — `jj git fetch` は `remote.<name>.tagOpt` を読むため設定なしだと tag が来ない。`jj git import` を併用して bare git 側で fetched された tag を jj 側にも反映する。両者を best-effort (`|| true`) にして、ネットワーク不通や設定不備でも後続処理が走れるようにする
- git: `git fetch --tags origin`

このように iface に `abstract` を追加すると `extends "iface.pkl"` する全実装 (jj/git/auto) で実装漏れが評価時エラーになる (DR-0001 の型システム恩恵)。外部 consumer には影響なし。

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

## SemVer group — `semver/check-bumped.pkl` + `semver/compare.pkl`

### `semver:check-bumped` — bump 漏れ gate

「比較対象 ref 以降に `triggerPaths` が変わったのに**利用側プロジェクトの version ファイル** (`versionFiles` で指定、e.g. `VERSION` / `Cargo.toml` / `package.json` / 利用者カスタム) が bump されていなければ fail」を検査するゲート Task。push 前 (`compareRef = main@origin`) や CI release 前 (`compareRef = 直近の v* tag`) のガードとして使う。

パラメータ化されており、利用側で複数インスタンスを切れる:

- `compareRefCmd: String` — 比較対象 ref を返す bash command substitution の中身
- `triggerPaths: List<String>` — 変更検知対象 (default `src/`)
- `versionFiles: List<String>` — bump 対象の version ファイル一覧 (利用側プロジェクトの `VERSION` / `Cargo.toml` / `package.json` 等、複数可)
- `taskName: String` — task 名 (利用側で `semver:check-version-bumped` / `semver:check-against-latest-release` のように別名を付ける)
- `needsFetchTags: Boolean` — true なら deps に `vcs.fetchTags` を挟む (v0.0.9 追加)。tag 系の compareRef (`git tag -l 'v*' ...`, `vcs:latest-tag()`) を使う場合に jj/git の tag 同期を保証するため。default false (push 前 gate 等、`main@origin` 比較では不要)

`(semverCheck) { ... }.check` の object-amends で利用側が 2 task に展開する流儀。実装は `vcs.diffSummary` で trigger paths の差分を取り、空でなければ `bump-semver compare gt VERSION vcs:"$compare_ref":VERSION` で SemVer 比較する。

### `semver:compare` — ad-hoc CLI ラッパ (v0.0.8)

`bump-semver compare` の薄いラッパで、pkfire の `acceptsArgs = true` 経由で任意の `<OP> <INPUT> <INPUT>` を渡す:

```
pkf run semver:compare -- gt VERSION 1.0.0
pkf run semver:compare -- lt Cargo.toml vcs:latest-tag():Cargo.toml
```

pkfire の `Task.deps: Listing<Task>` は引数渡し未対応なので、本 task は **ad-hoc CLI 利用専用**。`semver:check-bumped` のような複合 gate は本 task を deps として再利用できず、内部で `bump-semver compare` を直接呼ぶ別 task として維持する。

### bump-semver 必須 — SemVer 比較 fallback は意図的に未実装

`bump-semver` CLI が無ければ `not implemented: install bump-semver` メッセージで停止する。pure-Pkl / bash の fallback (`sort -V` 等) は実装しない:

- SemVer は prerelease (`-rc.1`) / build metadata (`+sha.abc`) の比較ルールが複雑で、`sort -V` は `0.14.0-rc.1` と `0.14.0` の順序が直感に反する
- `bump-semver` は semver.org 準拠で実装されており、責務をそちらに集約する方が DRY
- pkf-tasks の責務は VCS dispatch + task 合成、SemVer 比較は bump-semver に任せる

利用側は `brew install kawaz/tap/bump-semver` 等で導入。CI 環境でも同様。`semver/*` は `vcs/*` に依存する一方向 layering で、逆依存はない。詳細は [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md)。

## migrate group — pkf-tasks 自身の version 追従 (v0.0.11+)

利用側 Taskfile.pkl の `pkf-tasks@<version>` import が古いまま放置されないよう、push 前の gate と手動更新の action を提供する。設計思想は **push の deps = 気づき発火点** で、`semver:check-*` と同じ流儀。

### gate と action の分離

- **`migrate:check-pkf-tasks-current`** (`migrate/check-current.pkl`) — gate task。利用側 Taskfile.pkl の `pkf-tasks@<version>` import が最新 release より古いと fail。`push` task の deps に置く想定
- **`migrate:update-pkf-tasks`** (`migrate/update-self.pkl`) — action task。gate が fail したときの復旧手段として利用者が手動で `pkf run migrate:update-pkf-tasks` 実行。`sed -i.bak` で in-place 書き換えするが **自動 commit はしない** (diff 確認は利用者責任)

gate と action を分けるのは `semver:check-version-bumped` (gate) と `bump-version` (action) と同じ思想。「pkf-tasks 古いと push で fail → 利用者が action 実行 → diff 確認 → commit」のループを成立させる。

### bump-semver 経由の最新 tag 取得 (v0.0.12 で統合)

v0.0.11 では繋ぎ実装として `git ls-remote --tags` + `awk | grep | sort -V | tail -1` の bash pipeline で remote の最新 tag を取得していた。v0.0.12 で bump-semver v0.15.0 が `vcs:latest-tag(<repo>)` をサポートしたので、**本来の責務分担** (bump-semver = VCS-aware SemVer 比較 + ref schema、pkf-tasks = task 合成) に整合させた:

- 最新 tag 取得: `bump-semver get vcs:latest-tag(kawaz/pkf-tasks)`
- 比較: `bump-semver compare ge <current> <latest>` — 文字列マッチではなく **SemVer 比較** (current >= latest なら gate pass。pre-release / build metadata も semver.org 準拠で比較)
- 利用側が未 release 版を pin している場合や RC release 中も適切に通る

これにより bash の複雑な pipeline が不要になり、knowledge の集約先 (VCS-aware SemVer ref schema = bump-semver、task 合成 = pkf-tasks) がクリーンに分離された。DR-0006 (vcs as knowledge storage の方針) と整合する責務移動。

## 配布 — Pkl package + GitHub Release

`tasks/PklProject` で package metadata を宣言:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "2.0.0"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

tag 命名は `pkf-tasks@<version>` (mizchi/pkfire の流儀踏襲、詳細は [DR-0003](./decisions/DR-0003-tag-naming.md))。`pkl project package` で zip + metadata json + SHA256 を生成し、GitHub Release に asset としてアップロード。`pkg.pkl-lang.org` proxy 経由で利用側の `pkl eval` / `pkf list` が取得する。

利用側の Pkl cache は `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/` で、SHA256 検証付きで保存される。一度取得すれば Pkl サーバが落ちていてもオフライン評価可能。

## CI / リリース自動化

`.github/workflows/ci.yml`: main へ push / PR 時に pkfire の composite action (`mizchi/pkfire@<commit SHA>`) で pkl + pkf を準備、各 module を `pkl eval` で smoke-test、`pkl project package` の生成可否を確認、自リポの翻訳ペアの整合性 (self dogfooding) を確認。

`uses:` を tag (`mizchi/pkfire@pkfire@<version>`) ではなく **commit SHA で pin** する。pkfire の release tag に `@` が含まれる (e.g. `pkfire@0.6.0`) ことが GitHub Actions の workflow parser を壊す (workflow file レベルで fail し log すら取れない) ため。SHA pin は副次効果として supply chain security 観点でも推奨される。詳細は [DR-0004](./decisions/DR-0004-pkfire-action-sha-pin.md)。

`.github/workflows/release.yml`: `pkf-tasks@*` tag が push されたら自動で `pkl project package` を実行し、生成された 4 asset を GitHub Release にアップロード。tag と PklProject の version が一致しない場合は fail させて誤 publish を防ぐ。

## 関連ドキュメント

- 設計判断記録: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- 変更履歴: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* 共通の docs 構造規約: `~/.claude/rules/docs-structure.md`
