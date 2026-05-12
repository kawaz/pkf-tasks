# `docs/` — ドキュメント整合性チェック

> [English](./README.md) | 日本語

kawaz `docs-structure.md` ルールに基づく翻訳ペア (`*-ja.md` / `*.md`) の存在 / 相互リンク / commit timestamp 順序を一括検証する。`push` の deps に組み込むことで「英訳忘れ」「先に英訳を更新して原本 (-ja) を放置」を push 時に検知する想定。

1 Task で bash の for ループ (Listing<Task> 展開しない) で実装。cache 独立粒度が要らない処理は 1 Task で十分という pkfire 流儀。VCS の commit timestamp 取得は内部 bash 関数で jj / git 自動切替 (`tasks/vcs/auto.pkl` と同じ実行時判定方針)。

## モジュール構成

- `translations.pkl` — 翻訳ペア整合性チェックの Task と、利用側で別ペアセットを検証したい場合の `task(pairs)` 関数を export

## 提供 task

### `docs:check-translations`

- 便利度: ★★★
- 内部動作: 各ペア `<name>` について以下を順に検査
  1. `<name>-ja.md` が存在しなければ skip (任意ペア対応)
  2. `<name>.md` (英語版) が同じディレクトリに存在することを確認
  3. `<name>-ja.md` の先頭 5 行に `> [English](./<basename>.md) | 日本語` の固定リンク (`grep -qF` 完全一致)
  4. `<name>.md` の先頭 5 行に `> English | [日本語](./<basename>-ja.md)` の固定リンク
  5. commit timestamp 順序: `ja_ts <= en_ts` (英訳が遅れて push されてないことを確認)
- デフォルト対象ペア: `README` / `docs/DESIGN` / `docs/MANUAL` (kawaz docs-structure.md 慣習)
- 利用例: `push` の deps に組み込む

```pkl
tasks {
  ...kawaz.docs.allTasks   // docs:check-translations を登録
}
```

ペアを上書きしたい場合は `translations.pkl` の `task(pairs)` 関数を使って別 Task を作る:

```pkl
import "package://...pkf-tasks@0.0.16#/docs/translations.pkl" as translations
local myCheck = translations.task(new Listing<String> {
  "README"
  "docs/CONTRIBUTING"
})
tasks { myCheck }
```

- バグ history: 0.0.4 で `for` ループの exit code が個別 pair の fail を伝播せず常に成功扱いだった問題を `set -euo pipefail` + `failed` accumulator で fix。`for` 内で `|| { ...; failed=1; }` し最後に `exit $failed` する形に変更
- 設計判断: commit timestamp 取得を stat mtime ではなく VCS log にしているのは、jj workspace 切り替えで stat mtime が揺れるため。jj は `jj log -T 'committer.timestamp().format("%s")'`、git は `git log -1 --format=%ct -- <file>`

## 関連

- [上位 README](../../README-ja.md)
- `~/.claude/rules/docs-structure.md` — 翻訳ペア命名規約とリンクテンプレートの定義元
- 関連 module: `tasks/vcs/auto.pkl` の jj/git 実行時判定パターンを内部で踏襲
