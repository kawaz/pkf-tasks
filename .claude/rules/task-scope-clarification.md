# task 名を語るときのスコープ明記ルール

pkf-tasks には task が 2 レイヤあるので、会話で task 名を出すときは **必ずどちらのレイヤか明記する**。

## 2 レイヤ

| レイヤ | パス | 性格 | 例 |
|---|---|---|---|
| ライブラリ task | `tasks/<group>.pkl` 配下 | kawaz/* 各リポが `import` して使う再利用 task | `vcs:commit` / `docs:check-translations` / `semver:check-bumped` |
| Taskfile.pkl 内 task | repo root の `Taskfile.pkl` | pkf-tasks 自身の dogfood / CI 用 inline task | `ci` / `lint:pkl` / `lint:all-coverage` |

## 書き方

- NG: 「lint:pkl を消す」「lint の整理」
- OK: 「**Taskfile.pkl 内の** lint:pkl を消す (tasks/ 配下には影響なし)」
- OK: 「**tasks/vcs.pkl の** vcs:commit を改名する」

整理案・diff 提示・名前変更の議論で特に重要。同じ task 名でもレイヤが違えば全くの別物。
