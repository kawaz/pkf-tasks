# DR-0007: pkf-tasks の group 構造規約

- Status: Active
- Date: 2026-05-12

## Context

v0.0.7 までは `vcs/` と `docs/` の 2 group だけだったが、v0.0.7 〜 v0.0.13 で `lint/` `semver/` `migrate/` の 3 group が加わり、計 5 group になった。各 group の責務境界と命名・集約パターンは経験的に揃ってきているが、暗黙の規約のままだと新 group 追加時にブレが出る。

新 group を追加する判断、既存 group への機能追加、`all.pkl` 集約スタイル、task 命名 (`<group>:<action>`) を **明示的なルール** として記録する。これは DR-0006 (vcs/* を VCS knowledge 集積場として位置付ける) の上位概念に近い: pkf-tasks 全体の構造ガイドライン。

## Decision

### 1. group の単位 = 1 ドメイン責務

各 group は **1 つのドメイン責務** に対応:

| group | ドメイン |
|---|---|
| `vcs/` | jj/git の操作 (auto-dispatch + knowledge 集積 — DR-0006) |
| `docs/` | ドキュメントの整合性 (現状は翻訳ペア検証のみ) |
| `lint/` | 言語横断 lint (`lint:pkl`) + メタ lint (`lint:all-coverage`) |
| `semver/` | SemVer の比較 + gate (`check-bumped`, `compare`) |
| `migrate/` | pkf-tasks 自身の version 追従 gate + action |

判断基準:
- 新 task が既存 group のドメインに収まる → 既存 group に追加
- 新 task が独立した責務を持つ → 新 group を作る
- 「VCS 不問の操作」「特定 release flow に固有」など 1 行で説明できないなら group 化はまだ早い (将来検討)

### 2. 命名規則

#### Task 名: `<group>:<action>`

- 例: `vcs:commit` / `lint:pkl` / `semver:check-bumped` / `migrate:check-pkf-tasks-current`
- pkfire の `Task.name` regex `^[a-zA-Z][a-zA-Z0-9_:./-]*$` に従う
- `:` 区切りで group / action を明示 (`pkf list` でソートしたとき group 単位でまとまる)
- 利用側で複数 instance 化する場合は `<group>:<action>-<modifier>` で枝分かれ (例: `semver:check-version-bumped` / `semver:check-against-latest-release`)

#### group ディレクトリ名: 小文字、単数形か明確な複数形

- 例: `vcs/` `docs/` `lint/` `semver/` `migrate/`
- 既存規約: `docs/` だけ複数形 (`docs/translations.pkl` のような複数翻訳ペアを扱う想定で許容)、それ以外は単数形

### 3. group 内の module 構造

各 group は以下のファイルを持つ (必要に応じて):

```
tasks/<group>/
├── <action>.pkl          各 task の実装
├── iface.pkl             abstract module (vcs のみ、複数実装が必要な場合)
├── jj.pkl / git.pkl /    iface 実装 (vcs のみ)
│   auto.pkl
└── all.pkl               sub namespace 集約 (v0.0.10+)
```

#### `iface.pkl` を作るべきか

- **作る**: jj/git のように **複数実装が並存** し、利用者が dispatch されたものを使う場合 (vcs/* の唯一の例)
- **作らない**: 単一実装で完結する場合 (docs / lint / semver / migrate)

### 4. `all.pkl` 集約スタイル

v0.0.10 で導入。sub namespace の集約 endpoint:

```pkl
// tasks/<group>/all.pkl
module com.github.kawaz.pkfTasks.<group>.all

import "@pkfire/Taskfile.pkl"  // allTasks の型に必要

import "<action1>.pkl" as action1Mod
import "<action2>.pkl" as action2Mod

// parameterize 対象 = module 参照で公開 (利用側で (...).check のように amends)
<action1> = action1Mod

// ad-hoc 用 task = task 直接公開 (利用側でそのまま登録)
<action2> = action2Mod.<taskField>

// 一括登録用 (利用側で `tasks { ...kawaz.<group>.allTasks }`)
allTasks: Listing<Taskfile.Task> = new {
  // module 参照のものは含めない (利用側で parameterize して instance task を作る)
  <action2>  // task 直接公開のもののみ含める
}
```

選択基準:
- **module 参照** で公開: 利用側で `(...) { compareRefCmd = ... }.check` のように **parameterize** する想定の module (例: `semver.checkBumped`)
- **task 直接** で公開: parameterize 不要で **そのまま使う** ad-hoc task (例: `semver.compare`, `lint.pkl`, `migrate.check`)

### 5. root `all.pkl` (`tasks/all.pkl`)

各 group を `kawaz.<group>` namespace に集約:

```pkl
module com.github.kawaz.pkfTasks.all

import "<group>/all.pkl" as <group>Mod   // sub all.pkl 経由
// または直接 import (sub all.pkl 不要な単純 group の場合):
// import "<group>/<file>.pkl" as <group>Mod

<group> = <group>Mod
```

利用側は `import "package://.../all.pkl" as kawaz` 1 行で `kawaz.<group>.<action>` 形式で全 task にアクセス。

### 6. 孤児検出 (`lint:all-coverage`)

`tasks/` 配下の全 module は他のどこかから参照されていることを **`lint:all-coverage` で強制**。新規 module 追加時に自動検出され、`tasks/<group>/all.pkl` に re-export を加えるか、別 module から import される。これは規約の一部 = 「孤児 module は規約違反」。

## Rationale

### なぜ group という単位で切るのか

- 利用側で「pkf-tasks の vcs 機能だけ欲しい / docs 機能だけ欲しい」のような **部分採用** を可能にする (`import ".../vcs/auto.pkl" as vcs` で 1 group だけ取り込み可)
- `pkf list` でソートしたとき同 group の task が隣接して表示される (`<group>:<action>` 命名)
- 責務境界が明示されることで「この group は何のためにあるか」が自己説明的になる

### なぜ task 命名で `<group>:<action>` か

- pkfire の Task.name は `:` を許容しており、`vcs:commit` / `lint:pkl` のような構造化命名が pkf list で読みやすい
- 利用側の Taskfile.pkl でも `tasks { kawaz.vcs.commit }` のように namespace 参照と一致して可読性が高い

### なぜ sub `all.pkl` を作るのか

v0.0.10 で root `all.pkl` を導入した時点では、root から各 module を直接 import していた。v0.0.11 で sub `all.pkl` を導入したのは:

- group 内 module が複数になった (`semver/` の `check-bumped` + `compare`、`lint/` の `pkl` + `all-coverage`) ため、root から個別 import すると root all.pkl が肥大化
- sub 集約で **group ごとの内部構造を root から隠蔽** できる (root は `kawaz.semver` の中身を知らなくていい)
- 各 group 内で「module 参照 vs task 直接」の使い分けを sub all.pkl 内で決められる

### 不採用案

**1. group なしのフラット構造**

`tasks/commit.pkl` `tasks/push.pkl` `tasks/check-bumped.pkl` のように全 module を `tasks/` 直下に置く案。**不採用**: 100+ module になったら破綻、責務境界が不明、命名空間衝突のリスク (`vcs:commit` と「commit hash の何か」が混ざる等)。

**2. 1 group = 1 module の徹底**

`vcs/` を `vcs.pkl` (単一ファイル) にする案。**不採用**: jj.pkl / git.pkl / auto.pkl のような実装分離が困難、Pkl の module-per-file 設計とも合わない。

**3. namespace を `<owner>.<package>.<group>` で正規化**

利用側で `import "package://...#/all.pkl" as kawaz` ではなく完全修飾名 (`com.github.kawaz.pkfTasks.<group>`) で参照する案。**不採用**: typing コストが大きい、Pkl の `import "..." as <short>` 慣用に反する。完全修飾名は module 内部の `module com.github.kawaz.pkfTasks.<group>` 宣言で表現されており、利用側は短縮 alias で十分。

## Consequences

- 新 group 追加時に本 DR を参照してチェックリストに沿う:
  1. ドメイン責務が 1 つに収まるか
  2. 既存 group に統合できないか
  3. 命名は `<group>:<action>` か
  4. sub `all.pkl` で集約するか (module が 2+ になるなら作る)
  5. `tasks/all.pkl` に namespace を追加
  6. `lint:all-coverage` が通るか
- 既存 group への機能追加も、同じ命名 + 集約パターンに従う (既存 module を真似る)
- 将来「group が 10+ になった」時の再分割は別 DR で扱う (例: ecosystem 別の sub-package 化)

## 関連

- 上位 DR: DR-0006 (vcs/* を VCS knowledge 集積場として位置付け) — group 単位の責務記述の最初の例
- 集約スタイル導入: v0.0.10 (`all.pkl`)、v0.0.11 (sub `all.pkl` + `allTasks`)
- 命名規約の根拠: pkfire の `Task.name` regex
- 部分採用パターン: `kawaz/bump-semver` の Taskfile.pkl (`all.pkl` 経由) / 古い消費者の個別 import (`docs/translations.pkl` のみ等)
