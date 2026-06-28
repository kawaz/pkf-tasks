# kawaz.semver.checkBumped が versionFile compareRef 不在で失敗 (新規 VERSION ファイル / リポ rewrite 起点)

- Status: Open
- Date: 2026-05-22
- Priority: Middle (initial-release / 大きな main rewrite で必ず詰まる)
- 発見元: kawaz/hyoui Rust 一本化フェーズ (2026-05-22、`docs/journal/2026-05-22-rust-rewrite.md`)
- 姉妹 issue: `2026-05-21-checkbumped-initial-push-fails.md` と関連、別パターン

## 症状

`main@origin` 自体は存在するが、その compareRef 時点で **versionFile (例: VERSION) が存在しない**場合、`kawaz.semver.checkBumped` 内の `bump-semver compare gt 'VERSION' "vcs:main@origin:VERSION"` が `No such path: VERSION` で失敗し、push が通らない。

```
bump-semver: jj file show -r main@origin VERSION: Error: No such path: VERSION
ERROR: VERSION not bumped since main@origin
```

## 再現条件

- VERSION ファイルを **新規追加**して initial release する (compareRef 側に VERSION が無い)
- main を別系統に rewrite (今回の hyoui のケース: poc 全捨て → Initial empty commit から作り直し → remote main が空 commit に巻き戻る → 新 VERSION 追加して push)
- 既存 versionFile を削除→別系統で再追加した直後

## 既存姉妹 issue との違い

- `2026-05-21-checkbumped-initial-push-fails.md` = **compareRef (main@origin) ブランチ自体不在**、`jj diff` が失敗
- 本 issue = **compareRef ブランチは存在するが versionFile がその時点で不在**、`bump-semver compare gt` が `jj file show` 段階で失敗

両者は別の boundary で発生するが、根は同じ「compareRef 側に比較対象がない初回」未対応。

## 回避策 (hyoui 側で実施)

Taskfile.pkl の push deps から `checkVersionBumped` を一時除外して `pkf run push` 実行 → push 完了後 `jj restore --from @- Taskfile.pkl` で取り消し (一時編集は @ にのみ残し commit しない)。実装記録は kawaz/hyoui `docs/journal/2026-05-22-rust-rewrite.md` のハマり所表に記載。

## 修正案

- 案 A: `bump-semver compare gt FILE_A FILE_B` で **FILE_B が「存在しないファイル」を指す場合は gt=true** を返す (= 新規ファイル = 必然的に bump 済扱い)。`bump-semver` 側の修正。
- 案 B: `kawaz.semver.checkBumped` 内で各 versionFile について「compareRef 側に存在するか」を先に確認し、不在なら新規追加扱いで skip (= bump 不要)。`pkf-tasks` 側の修正。

案 A の方が `bump-semver` の責務として自然 (compareRef 側不在 = 比較不能 = ローカル側が新しい)。

## 関連

- kawaz/hyoui の `docs/journal/2026-05-22-rust-rewrite.md` (回避策実装記録)
- 姉妹 issue: `2026-05-21-checkbumped-initial-push-fails.md`
- 元 GitHub Issue: https://github.com/kawaz/pkf-tasks/issues/1 (本 docs/issue/ への転記後 close)
