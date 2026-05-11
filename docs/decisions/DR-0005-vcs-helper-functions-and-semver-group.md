# DR-0005: vcs/* に Pkl helper function を追加し、bump-semver 依存の semver グループを新設する

- Status: Active
- Date: 2026-05-11

## Context

`tasks/vcs/{iface,jj,git,auto}.pkl` は当初 **Task only export** で設計された (`commit` / `push` / `fetch` / `ensureClean` の 4 つの abstract Task)。各実装側が Task を埋めるシンプルな構造。

しかし利用側で「push 前に `src/` が変わってたら VERSION も bump されてるかチェック」のような **higher-level なゲート task** を書きたい場合、低レベル primitive として:

- 任意の ref と paths の diff を取る (jj/git dispatch)
- 任意の ref でのファイル内容を取る (jj/git dispatch)

が必要になる。これらは Task として export するより、bash の `$(...)` で展開する **文字列断片を返す Pkl function** として提供するのが自然 (他 Task の cmd 内に埋め込める)。

加えて、bump-semver は kawaz の汎用 SemVer 比較 CLI で、`bump-semver compare gt VERSION vcs:main@origin:VERSION` のような VCS-aware な比較もサポートしている。kawaz の他リポでも `check-version-bumped` のような task を justfile で持っており、これを pkfire / Pkl の世界でも共通化したい。

## Decision

### 1. vcs/auto.pkl に Pkl function helper を追加

以下を **task export と並列に function として export**:

- `function diffSummary(ref: String, paths: List<String>): String` — `$(if jj root...; jj diff -r ...; elif git ...; fi)` を返す
- `function readAtRef(ref: String, path: String): String` — `$(if jj root...; jj file show ...; elif git show; fi)` を返す

これらは利用側で `\(vcs.diffSummary("$compare_ref", List("src/")))` のように Pkl 文字列補間で他 Task の cmd 内に埋め込まれる。Pkl 評価時に jj/git dispatch を含む bash 断片に展開され、実行時に bash が if-else を評価する (auto-detect)。

### 2. semver グループ (`tasks/semver/check-bumped.pkl`) を新設

bump-semver CLI に依存する高レベル task を提供する新カテゴリ。`tasks/vcs/` が VCS 抽象、`tasks/docs/` がドキュメント整合性チェック、`tasks/lint/` が言語横断 lint と同列に、`tasks/semver/` を「SemVer 関連のゲート」として切る。

命名検討: 当初は `tasks/bump/` (action 名) を候補にしたが、`bump:` だと「bump-semver CLI 名」「version bump アクション」の両方を指して意味が狭い (将来 `bump:compare` のような比較系を入れると変)。`semver:` (ドメイン名) なら `semver:check-bumped` / `semver:check-against-latest-release` / 将来の `semver:compare` / `semver:bump-version` 等を一貫して収容できる。pkf-tasks の他グループ (`vcs:` / `docs:` / `lint:`) もドメイン名なので、流儀として揃う。

パラメータ化:
- `compareRefCmd: String` — 比較対象 ref を返す shell command substitution の中身
- `triggerPaths: List<String>` — 変更検知対象
- `versionFiles: List<String>` — bump 対象 VERSION ファイル一覧
- `taskName: String` — task 名 (利用側で 2 つインスタンス化する時に別名を付ける)

利用側で push-time check と CI release-time check を 1 module から 2 task に展開できる。

## Rationale

### 不採用案

**1. vcs/helpers.pkl を別 module として切り出す**

`vcs/auto.pkl` を「Task only」のまま保ち、helper function は `vcs/helpers.pkl` に分離する案。**不採用**: jj/git dispatch ロジックを auto.pkl の `autoCmd` と helpers.pkl で **二重に持つ** ことになる。共通基盤を抽出すると iface.pkl / jj.pkl / git.pkl / auto.pkl / helpers.pkl の 5 module 構造になって過剰。auto.pkl が「auto-detect なら何でもここに集約」のシングルゲートウェイになる方が利用側の import も 1 つで済む。

**2. bump-semver なし環境用の pure-Pkl/bash fallback を実装する**

`vcs.readAtRef` で旧 VERSION を取り出し、`sort -V` で簡易 SemVer 比較する fallback。**不採用**: SemVer は prerelease (`-rc.1` 等) や build metadata (`+sha.abc`) の比較ルールが複雑で、`sort -V` では `0.14.0-rc.1` と `0.14.0` の関係が直感に反する結果になる (`-rc.1` が後とみなされる)。bump-semver は SemVer 仕様 (semver.org) を正確に実装しており、ここで bash で再実装する価値は薄い。**「できないものはスパッと切る」** — bump-semver が無ければ `not implemented: install bump-semver` メッセージで止める。

**3. semver グループを作らず、利用側で各々書く**

`tasks/semver/check-bumped.pkl` を提供せず、各 repo の Taskfile.pkl で 30 行の cmd を直書きする案。**不採用**: kawaz の複数 repo (bump-semver, claude-cmux-msg, 他) で同じパターンを繰り返すことになる。pkf-tasks が共通化の本拠地である以上、ここに集約しない理由がない。

### 設計上のポイント

#### bump-semver の `vcs:` ref schema と pkf-tasks の vcs helpers の責務重複

bump-semver CLI には `bump-semver compare gt VERSION vcs:main@origin:VERSION` のような VCS-aware な参照スキームがあり、内部で jj/git auto-detect している。pkf-tasks の `vcs.readAtRef` / `vcs.diffSummary` と機能的に重複する。

**この重複は許容する**:

- bump-semver は **単体ツール**としての価値 (Pkl/pkfire 不要で動く)。bump-semver 利用者全員が pkf-tasks を使うわけではない
- pkf-tasks の vcs helpers は **bump-semver 以外の task** でも使える汎用 primitive (将来 lint / format などの ref 比較系 task で活用余地)
- 棲み分けが成立しており、片方が消えても他方が生き残れる

DRY 違反のリスクとして「両者の jj/git dispatch ロジックが乖離する」可能性があるが:
- pkf-tasks 側: `jj root` / `git rev-parse --git-dir` で親方向探索
- bump-semver 側: 同じ親方向探索パターンを採用する想定 (実装責任は bump-semver)

両者ともに jj-workflow.md / git-workflow.md の流儀に従う限り齟齬は生じない。

#### Pkl function を「コマンド文字列を返す」用途で使う流儀

`diffSummary` / `readAtRef` は値 (構造化データ) ではなく **bash 断片** を返す。Pkl の用途としては変則的だが:

- Pkl は文字列補間と triple-quoted string が強力で、bash 断片の組み立てに向く
- 利用側は cmd 内に `\(vcs.diffSummary(...))` で埋めるだけ。pkfire の Task ではなく Pkl の文字列処理として完結
- 型安全 (引数の型は List<String> / String で強制)、リテラル文字列の typo を Pkl 評価時に検出できる

この流儀は他の task module (docs/translations.pkl 等) には現状適用していない。必要に応じて後続 DR で同様の helper 追加を検討する。

## Consequences

- vcs/auto.pkl の役割が「Task export」+「function export」の二層になる。README / DESIGN の Design セクションで明示する
- 利用側は `import "package://.../vcs/auto.pkl" as vcs` 1 つで Task と helper の両方にアクセスできる
- semver グループは bump-semver CLI 必須。CI 環境では `brew install kawaz/tap/bump-semver` や同等の手段で bump-semver を入れる前提
- semver グループのパラメータ化 (`compareRefCmd` / `triggerPaths` / `versionFiles` / `taskName`) は利用側で `(semverCheck) { ... }.check` の object-amends pattern で上書きする。pkfire の他 task module と同じ流儀
- 将来 `tasks/semver/` 配下に他 task (e.g. `semver:bump-version`, `semver:compare`) を追加する余地を残す
- semver/* は vcs/* に依存する一方向の layering (`semver → vcs`)。逆依存はなく、責務オーバーラップの懸念は依存グラフ上 clean に整理できる

## 関連

- 実装: `tasks/vcs/auto.pkl` の `diffSummary` / `readAtRef` function、`tasks/semver/check-bumped.pkl`
- 利用側: `kawaz/bump-semver` の `Taskfile.pkl` で `semver:check-version-bumped` (push 前) と `semver:check-against-latest-release` (CI release 前) の 2 task をインスタンス化
- 上位 DR: DR-0002 (Pkl は FS state を見ないため runtime dispatch、本 DR の helper function はその応用)
- bump-semver: `kawaz/bump-semver` の `compare` subcommand + `vcs:<ref>:<path>` ref schema
