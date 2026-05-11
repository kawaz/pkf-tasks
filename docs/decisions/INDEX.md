# Decision Records (DR) Index

kawaz/pkf-tasks の設計判断記録一覧。ファイル名は `DR-NNNN-title.md` (4 桁ゼロパディング)。`docs-structure.md` ルールに従い `## Active` / `## Archived` / `## Moved to research/` で区分する。

## Active

- [DR-0001](./DR-0001-abstract-module-extends.md) — abstract module を `extends` で実装する流儀 (`amends` ではなく)
- [DR-0002](./DR-0002-runtime-vcs-dispatch.md) — VCS 検出を Pkl 評価時ではなく cmd 内実行時に行う (Pkl の pure-eval 制約への対応)
- [DR-0003](./DR-0003-tag-naming.md) — tag 命名は `pkf-tasks@<version>` (pkfire 流儀踏襲、`v<version>` ではない)
- [DR-0004](./DR-0004-pkfire-action-sha-pin.md) — GitHub Actions composite action 参照は commit SHA で pin する (pkfire の `@@` 2重 ref 問題への対応)
- [DR-0005](./DR-0005-vcs-helper-functions-and-semver-group.md) — vcs/* に Pkl helper function (`diffSummary` / `readAtRef`) を Task export と並列に追加、bump-semver 依存の `semver:` グループ新設

## Archived

(なし)

## Moved to research/

(なし)
