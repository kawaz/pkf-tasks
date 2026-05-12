# DR-0008: lint group を library export から外し Taskfile.pkl に inline 化 (v2.0.0)

- Status: Active
- Date: 2026-05-12
- Supersedes: DR-0007 の「`lint:all-coverage` を library task として export する」前提 (DR-0007 自体は構造規約として有効、lint 関連の記述だけ無効化)

## Context

v1.0.0 (2026-05-12 リリース) で `tasks/<group>.pkl` flat 化を含む public API stable 宣言を行った直後、kawaz から「他にも汎用じゃないのに汎用ヅラしてる task がないか監査」の指示。`lint` group (`tasks/lint.pkl` + `tasks/lint/pkl.pkl` + `tasks/lint/all-coverage.pkl`) を再評価したところ:

- `lint:pkl` (Pkl format -w) は **pkf-tasks 自身の dogfood** が主用途。利用側 (`kawaz/bump-semver`) は自前の `kawaz.lint.pkl` を `push` deps から呼ぶだけで、必要なら 5 行で自前定義できる
- `lint:all-coverage` は **library 開発者用** の孤児 module 検出 gate。pkf-tasks 自身の `tasks/` ディレクトリ構造を検査するもので、利用側プロジェクトの `tasks/` を検査する用途は存在しない
- どちらも「library として export する正当性 = 複数の kawaz リポで再利用される」基準を満たさない

v1.0.0 公開から 24 h 以内、唯一の consumer (`kawaz/bump-semver`) も `@1` pin を本格運用していない状態だったため、v1 を破壊して v2.0.0 に上げる判断:

- SemVer 厳守なら minor 破壊は不可、major bump 一択
- 利用者 1 件 (kawaz 内部) のうちにやり切るのが将来コスト最小

## Decision

### library 側 (`tasks/`): lint group を完全削除

- 削除: `tasks/lint.pkl` / `tasks/lint/pkl.pkl` / `tasks/lint/all-coverage.pkl` / `tasks/lint/README.md` / `tasks/lint/README-ja.md`
- `tasks/all.pkl` から `kawaz.lint.*` namespace を削除 (`import "lint.pkl" as lintMod` 行も削除)
- `tasks/lint/` ディレクトリごと削除

これにより v2.x の public API contract から `lint` group が消える (1.x で contract に含めていた `tasks/lint.pkl` FQN・task 名・`kawaz.lint.*` namespace すべて廃止)。

### dogfood 側 (`Taskfile.pkl`): inline 化 + リネーム

`pkf-tasks` 自身は `pkl format -w` と孤児検出を引き続き必要とするので、library export から外した同等品を repo-root `Taskfile.pkl` に **inline 定義** する:

- `lint:pkl` (Pkl format) — Taskfile.pkl 内の `local lintPkl: Task = new { ... }` として inline 定義
- `check:orphan-modules` (旧 `lint:all-coverage` をリネーム) — 孤児 module 検出。`check:` prefix で `semver:check-*` / `migrate:check-*` の gate-task 命名規約に揃える
- `lint` umbrella task は廃止 (`ci` task が直接 `lintPkl` / `checkOrphanModules` を deps に取る)

これらは **Taskfile.pkl 内 task** であり、library export ではない (consumer 側からは参照不可)。

## Rationale

### library export 基準

「library として export する正当性」の判定基準を本 DR で明示化:

- **再利用性**: 複数の kawaz リポで使われる、または将来使われ得る
- **抽象度**: 「kawaz/* 共通のワークフロー」として 1 行で説明できる責務単位
- **使い回し可能なシグネチャ**: parameterize 不要、または明示的に parameterize スロットを提供

`lint:pkl` は再利用性 ○ (どのリポでも Pkl format したい) だが、抽象度の観点で「`pkl format -w` を呼ぶだけの 5 行 task」を library 化する価値が薄い。`lint:all-coverage` は pkf-tasks 自身の構造 (`tasks/<group>/<file>.pkl`) を前提にしており、汎用 library として export できない (consumer 側に同じ構造が無い)。

### `check:` prefix へのリネーム

`lint:all-coverage` という名前は「all-coverage = なんの coverage か不明 (テストカバレッジ?)」「lint = コード整形と混同」の 2 重に分かりにくかった。Taskfile.pkl 内に降ろすタイミングで:

- 対象が明示される名前 → `orphan-modules` (孤児 module の検出)
- 既存規約に揃える → `check:` prefix (semver:check-* / migrate:check-* と同列の構造 gate 系統)

結果 `check:orphan-modules` = 「`tasks/` 配下の孤児 module を検出する gate」と name だけで読める。

### v1.0.0 を 1 日で破壊する判断

DR-0003 (tag 命名) と同様に、SemVer 厳守は文化として確立済み。1.x stable 宣言から 24 h で 2.x に上げるのは大袈裟に見えるが:

- v1.x を「lint group が library に残った歴史的 release」として塩漬けにする方が長期コスト大
- 唯一の consumer (`kawaz/bump-semver`) は同セッション内で 2.0.0 に追従可能
- 公開直後の「修正可能なタイミング」を逃すと、将来 v2 まで lint group を背負うことになる

「stable 宣言 = 全ての設計判断が完璧」ではない。stable 宣言後でも明確な漏れが見つかったら major bump で誠実に修正するのが SemVer 流儀。

## Consequences

### 即時影響

- v2.0.0 release: `tasks/lint.pkl` 削除、`tasks/all.pkl` 更新、PklProject `version = "2.0.0"`
- pkf-tasks 自身の `Taskfile.pkl`: import URI `@1.0.0` → `@2.0.0`、`local lintPkl` / `local checkOrphanModules` を inline 定義、`ci` deps 更新
- consumer (`kawaz/bump-semver`): `kawaz.lint.allTasks` 削除、import URI `@2.0.0` に bump (別 commit で追従)

### 規約への反映

- DR-0007 (group 構造規約) の「`lint` group」「`lint:all-coverage`」言及は本 DR で無効化される。DR-0007 自体は構造規約 (1 group = 1 ドメイン、`<group>:<action>` 命名、sub `all.pkl` 集約 → v1.0.0 で `tasks/<group>.pkl` flat 化) として有効
- 今後の library export 判断には本 DR の **「library export 基準」** (再利用性 + 抽象度 + 使い回し可能なシグネチャ) を適用する

### 教訓

stable 宣言の直前 / 直後にこそ「汎用ヅラ task の監査」を行う。stable 後の major bump はコストが上がるため、宣言タイミングで「library export = 再利用される task」基準を一度ふるい直す習慣を持つ。

## 関連

- DR-0007 (group 構造規約) — 本 DR で lint 関連の前提が一部無効化される
- v1.0.0 release (flat 化 + stable 宣言): CHANGELOG.md
- v2.0.0 release: CHANGELOG.md
- consumer 追従: `kawaz/bump-semver` の Taskfile.pkl
