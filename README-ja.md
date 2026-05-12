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
