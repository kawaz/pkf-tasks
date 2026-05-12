# Decision Records (DR) Index

kawaz/pkf-tasks の設計判断記録一覧。ファイル名は `DR-NNNN-title.md` (4 桁ゼロパディング)。`docs-structure.md` ルールに従い `## Active` / `## Archived` / `## Moved to research/` で区分する。

## Active

- [DR-0001](./DR-0001-abstract-module-extends.md) — abstract module を `extends` で実装する流儀 (`amends` ではなく)
- [DR-0002](./DR-0002-runtime-vcs-dispatch.md) — VCS 検出を Pkl 評価時ではなく cmd 内実行時に行う (Pkl の pure-eval 制約への対応)
- [DR-0003](./DR-0003-tag-naming.md) — tag 命名は `pkf-tasks@<version>` (pkfire 流儀踏襲、`v<version>` ではない)
- [DR-0004](./DR-0004-pkfire-action-sha-pin.md) — GitHub Actions composite action 参照は commit SHA で pin する (pkfire の `@@` 2重 ref 問題への対応)
- [DR-0005](./DR-0005-vcs-helper-functions-and-semver-group.md) — vcs/* に Pkl helper function (`diffSummary` / `readAtRef`) を Task export と並列に追加、bump-semver 依存の `semver:` グループ新設
- [DR-0006](./DR-0006-vcs-as-knowledge-storage.md) — vcs/* を「VCS dispatch + jj/git knowledge 集積場」として位置付ける運用方針
- [DR-0007](./DR-0007-group-structure-conventions.md) — pkf-tasks の group 構造規約 (1 group = 1 ドメイン責務、命名 `<group>:<action>`、v1.0.0+ で `tasks/<group>.pkl` flat 化)
- [DR-0008](./DR-0008-lint-group-removed-from-library.md) — lint group を library export から外し Taskfile.pkl に inline 化 (v2.0.0、library export 基準を明示)

## Archived

(なし)

## Moved to research/

(なし)
