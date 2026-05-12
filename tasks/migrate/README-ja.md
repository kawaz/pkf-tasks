# `migrate/` — upstream 追従ペア (gate + action)

> [English](./README.md) | 日本語

利用側 `Taskfile.pkl` の `pkf-tasks@<version>` import / `pkfire@<version>` amends が upstream の最新 release より古くなっているかを検知し、必要なら自動で書き換える group。**gate + action のペアで対象別に 2 セット** = 計 4 task。`bump-semver` CLI が PATH 必須 (`vcs:latest-tag(<repo>)` を使って最新 tag を取得)。

設計思想は `semver:check-*` と同じ「**push の deps = 気づき発火点**」: `check-*` task を `push` の deps に挟むと、利用側 project の依存 pin が遅れていたら push 時に fail し、`update-*` task で復旧する流れ。

## モジュール構成

- `check-current.pkl` — `migrate:check-pkf-tasks-current` (pkf-tasks の追従 gate)
- `update-self.pkl` — `migrate:update-pkf-tasks` (pkf-tasks の自動更新 action、sed in-place)
- `check-pkfire-current.pkl` — `migrate:check-pkfire-current` (pkfire の追従 gate)
- `update-pkfire.pkl` — `migrate:update-pkfire` (pkfire の自動更新 action、`pkf migrate` ラップ)
- `all.pkl` — sub namespace 集約 (`kawaz.migrate.checkPkfTasks` / `updatePkfTasks` / `checkPkfire` / `updatePkfire`)

## 提供 task

### `migrate:check-pkf-tasks-current` (pkf-tasks gate)

- 便利度: ★★★
- パラメータ (`check-current.pkl` の module 属性):
  - `taskfilePath: String` — default `"Taskfile.pkl"`
  - `remoteRepoSpec: String` — `bump-semver` の `vcs:latest-tag(<arg>)` に渡す。default `"kawaz/pkf-tasks"`
  - `tagPrefix: String` — default `"pkf-tasks@"` (DR-0003)
- 内部動作:
  1. `bump-semver get 'vcs:latest-tag(kawaz/pkf-tasks)'` で最新 tag を取得
  2. `taskfilePath` から `pkf-tasks@<x.y.z[-rc.1][+sha.abc]>` を SemVer 2.0.0 完全対応 regex で抽出
  3. `bump-semver compare ge <current> <latest>` が真なら ok、偽なら fail
- 利用例: `push` の deps に置く (`(vcs.push) { deps { ..., kawaz.migrate.checkPkfTasks } }`)

### `migrate:update-pkf-tasks` (pkf-tasks action)

- 便利度: ★★
- 内部動作: `bump-semver get` で最新 tag を取得後、`sed -i.bak -E` (BSD/GNU 両対応) で `taskfilePath` の `pkf-tasks@<version>` を最新版に書き換え。自動 commit はしない (利用者が diff を確認してから手動 commit)
- 注意: SemVer 2.0.0 (pre-release + build metadata) 対応 regex。複数箇所マッチは全置換

### `migrate:check-pkfire-current` (pkfire gate)

- 便利度: ★★★
- パラメータ: `remoteRepoSpec` default `"mizchi/pkfire"` / `tagPrefix` default `"pkfire@"` (他は `checkPkfTasks` と同じ)
- 内部動作: `migrate:check-pkf-tasks-current` と同構造。対象 URI が `amends` (pkfire) か `import` (pkf-tasks) かの違いだけなので、tag prefix の grep が同じ regex で動く
- 責務分担: pkfire 本体 (`amends`) と pkf-tasks (`import`) は別 task として並存させる。1 task で両方扱うと「どっちが原因で fail したか」が見えにくくなるため

### `migrate:update-pkfire` (pkfire action)

- 便利度: ★★
- 内部動作: `bump-semver get` で最新 tag 取得 → `pkf migrate --to=<latest> --file=<taskfilePath>` を呼ぶ
- 設計判断: 自前 sed 実装ではなく **`pkf migrate` (pkfire 0.6.0+ 内蔵)** を使う。pkfire 側は **Pkl eval 検証つき** で amends URI を書き換え、eval fail 時に **自動 revert** する。自前 sed より strict
- 注意: `pkf migrate` は **amends URI のみ** 対象。`import` URI には適用できない (pkf-tasks 側は別 task `update-pkf-tasks` で対応)

## 利用例 (group 全体)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.16#/all.pkl" as kawaz

tasks {
  ...kawaz.migrate.allTasks   // 4 task 全部登録
}

// push の deps に check-* を組み込む
local push = (kawaz.vcs.push) {
  deps {
    kawaz.vcs.ensureClean
    kawaz.migrate.checkPkfTasks   // pkf-tasks 古ければ fail
    kawaz.migrate.checkPkfire     // pkfire 古ければ fail
  }
}
tasks { push }
```

glob target (pkfire 0.6.0+) でまとめて実行:

```bash
pkf run 'migrate:check-*'    # 全 check 系を一括実行
pkf run 'migrate:update-*'   # check fail 時の救済 (全 update 系を一括)
```

## 命名規約と breaking changes

- **v0.0.16** で `kawaz.migrate.check` → `kawaz.migrate.checkPkfTasks` 等に **対称命名** へ rename
- v0.0.15 まで pkf-tasks 向けが暗黙 `check` / `update` だったが、`checkPkfire` / `updatePkfire` 追加で対象不明瞭になったため整理 (DR-0007)
- task name (`migrate:check-pkf-tasks-current` 等) は v0.0.15 で既に対称命名済み。今回は **Pkl 側の property 名** のみの rename

## 関連

- [上位 README](../../README-ja.md)
- DR-0003 tag 命名 (`pkf-tasks@<version>` / `pkfire@<version>`)
- DR-0007 group 規約 (sub namespace 集約 + 対称命名)
- DR-0019 bump-semver の `vcs:latest-tag(<arg>)` ref schema (本 group の最新 tag 取得で利用)
- 関連 CLI: [`kawaz/bump-semver`](https://github.com/kawaz/bump-semver) (`brew install kawaz/tap/bump-semver`) / [`mizchi/pkfire`](https://github.com/mizchi/pkfire) の `pkf migrate`
