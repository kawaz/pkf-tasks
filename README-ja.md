# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) 向けの共通タスクモジュール集。提供 group:

- **`vcs/`** — jj/git auto-dispatch な VCS primitive と knowledge 集積場 (DR-0006)。
  - Task: `vcs:commit` / `vcs:push` / `vcs:fetch` / `vcs:ensure-clean` / `vcs:fetch-tags`
  - Pkl function helper (他 Task の cmd 内に埋め込んで使う): `vcs.diffSummary(ref, paths)` / `vcs.readAtRef(ref, path)`

- **`docs/translations.pkl`** — 翻訳ペア整合性チェック。
  - Task: `docs:check-translations` — `*-ja.md` / `*.md` ペアの存在 / 相互リンク / commit timestamp 順序を検証

- **`lint/`** — 言語横断 lint + meta lint。
  - `lint:pkl` — `**/*.pkl` + `PklProject*` に対して `pkl format -w`
  - `lint:all-coverage` — パッケージ内の孤児 module 検出

- **`semver/`** — parameterized な SemVer gate と ad-hoc compare。`bump-semver` CLI が PATH に必要。
  - `semver:check-bumped` (object-amends で per-instance parameterize) — `triggerPaths` が `compareRef` 以降に変更されたのに VERSION bump がなければ fail
  - `semver:compare` — `acceptsArgs = true` で `bump-semver compare` をラップ (ad-hoc CLI 利用、例: `pkf run semver:compare -- gt VERSION 1.0.0`)

- **`migrate/`** — 利用側 `Taskfile.pkl` の version drift 検知 gate + 自動修復 action。2 upstream を 2 ペアずつ (check / update) でカバー。
  - **pkf-tasks** (`import` URI):
    - `migrate:check-pkf-tasks-current` — Taskfile.pkl の `pkf-tasks@<version>` import が最新 release より古いと fail
    - `migrate:update-pkf-tasks` — import を最新 tag に sed 書き換え
  - **pkfire** (`amends` URI、v0.0.15+):
    - `migrate:check-pkfire-current` — Taskfile.pkl の `pkfire@<version>` amends が最新 release より古いと fail
    - `migrate:update-pkfire` — `pkf migrate --to=<latest>` (pkfire 0.6.0+ 内蔵、eval 検証つき、fail 時は自動 revert) をラップ
  - `check-*` task を `push` の `deps` に挟むと「気づき発火点」として機能。

## タスク一覧

便利度の凡例: ★★★ = 多くの kawaz リポで必須・`push` の deps に組み込む系 / ★★ = 多用・特定用途で便利 / ★ = ad-hoc / 特殊用途 / 内部実装由来

### `vcs/` — jj/git auto-dispatch VCS primitive + knowledge 集積場

| Task | 便利度 | 用途 | 引数 / 備考 |
|---|---|---|---|
| `vcs:commit` | ★★★ | ワーキングコピーの変更をコミット (jj/git 自動切替) | param: `message` (必須) |
| `vcs:push` | ★★★ | リモートへ push (jj/git 自動切替) | `(vcs.push) { deps { ... } }` で extend して push 前 gate を組む |
| `vcs:fetch` | ★★ | リモートから fetch (jj/git 自動切替) | — |
| `vcs:ensure-clean` | ★★★ | working copy が clean か検査 (jj/git 自動切替) | `push` の gate に組み込む |
| `vcs:fetch-tags` | ★★ | tag 同期 (jj/git 自動切替) | `semver:check-against-latest-release` 等の前段 |

> 補足: 他 Task の cmd に文字列補間で埋め込む Pkl helper function (`vcs.diffSummary` / `vcs.readAtRef`) もあります。library 内部実装 + 上級利用者向け、詳細は [DESIGN-ja.md](./docs/DESIGN-ja.md) を参照。

### `docs/` — 翻訳ペア整合性

| Task | 便利度 | 用途 | 引数 / 備考 |
|---|---|---|---|
| `docs:check-translations` | ★★★ | 翻訳ペア (`*-ja.md` / `*.md`) 整合性検査 | `push` の deps に組み込む |

### `lint/` — 言語横断 lint + meta lint

| Task | 便利度 | 用途 | 引数 / 備考 |
|---|---|---|---|
| `lint:pkl` | ★★★ | 全 Pkl ファイルを `pkl format -w` で自動整形 | `push` の deps に組み込む |

> library 内部用の `lint:all-coverage` (孤児 module 検出) もあります、consumer 用途は稀。

### `semver/` — SemVer gate + ad-hoc compare (`bump-semver` CLI 必須)

| Task | 便利度 | 用途 | 引数 / 備考 |
|---|---|---|---|
| `semver:check-bumped` | ★★★ | version bump 漏れ gate (利用側プロジェクトの version ファイルが必要なときに上がっているか検査) | object-amends で per-instance parameterize、利用例は下記 |
| `semver:compare` | ★★ | `bump-semver compare` の薄ラッパ | 例: `pkf run semver:compare -- gt VERSION 1.0.0` |

`semver:check-bumped` の利用例:

```pkl
(kawaz.semver.checkBumped) {
  compareRefCmd = "echo main@origin"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")  // 利用側プロジェクトの version ファイル
  taskName = "semver:check-version-bumped"
}.check
```

### `migrate/` — upstream 追従ペア (`bump-semver` CLI 必須)

| Task | 便利度 | 用途 | 引数 / 備考 |
|---|---|---|---|
| **`migrate:check-*`** | — (glob, gate 系) | upstream 追従の検知 | `pkf run 'migrate:check-*'` で一括、`push` の deps 想定 |
| └ `migrate:check-pkf-tasks-current` | ★★★ | pkf-tasks の `import` が最新 release より古いと fail | — |
| └ `migrate:check-pkfire-current` | ★★★ | pkfire の `amends` が最新 release より古いと fail | — |
| **`migrate:update-*`** | — (glob, fix 系) | upstream 追従の自動修復 | `pkf run 'migrate:update-*'` で一括、check fail 時の救済 |
| └ `migrate:update-pkf-tasks` | ★★ | pkf-tasks の `import` を最新 tag に書き換え | 自動 commit なし |
| └ `migrate:update-pkfire` | ★★ | pkfire の `amends` を最新 tag に書き換え (eval 検証あり) | 自動 commit なし |

## 使い方

`Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.15#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
  // (kawaz.semver.checkBumped) { compareRefCmd = ...; ... }.check  // parameterize して instance 化
}
```

集約 endpoint `all.pkl` (v0.0.10+) で全 group に `kawaz` 1 namespace 経由でアクセス。個別 import (`import ".../vcs/auto.pkl" as vcs`) も従来どおり可。

確認は `pkf list` / `pkf list -v` / `pkf graph --format mermaid` で。最新版は [Releases](https://github.com/kawaz/pkf-tasks/releases)。0.0.x は不安定なので exact pin 推奨、`migrate:check-*` gate が pin の鮮度を保つ。

pkfire 0.6.0+ では CLI で **glob target** (`path.Match` 構文) が使え、`<group>:<action>` 命名と相性が良い:

```bash
pkf run 'lint:*'            # 全 lint:* task を topological 順で
pkf run 'semver:*'          # 全 semver: gate / utility を一括
pkf list 'migrate:check-*'  # check-* 系の migration gate だけ確認
```

詳細: [CHANGELOG](./CHANGELOG.md) / [docs/DESIGN-ja.md](./docs/DESIGN-ja.md) / [docs/decisions/](./docs/decisions/) / 実例 [kawaz/bump-semver](https://github.com/kawaz/bump-semver/blob/main/Taskfile.pkl)

## License

[MIT](./LICENSE)
