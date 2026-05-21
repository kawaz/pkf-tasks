# kawaz.semver.checkBumped が初回 push（main@origin 不在）で失敗する

- Status: Open
- Date: 2026-05-21
- Priority: Middle（新規リポの初回 push が必ず詰まる）
- 発見元: kawaz/hyoui のブートストラップ（一度も push していない新規リポ）

## 症状

一度も push していない新規リポで `pkf run push` を実行すると、push の deps に入る
`semver:check-version-bumped`（`kawaz.semver.checkBumped` を parameterize したもの）が
失敗し、初回 push が通らない。

## 原因

`kawaz.semver.checkBumped` の `diffSummary` 相当の生成コマンドが、おおむね次の形:

```sh
diff_out=$(if jj root >/dev/null 2>&1; then jj diff -r "main@origin..@" ...; fi)
```

- `compareRef`（既定 `main@origin`）がまだ存在しない（初回 push 前のリポ）と
  `jj diff -r "main@origin..@"` は **exit 1** で失敗する。
- この `jj diff` は `if ... fi` という **複合コマンド**の中にあり、それを
  command substitution `$(...)` で受けている。
- bash の `set -e` には「単純コマンドの command substitution を単一変数代入で
  受ける場合は exit を伝播しない」という緩和があるが、**この緩和は中身が単純
  コマンドのときだけ**。`if/fi` のような複合コマンドを包んだ command substitution は
  緩和対象外で、`set -e` が代入の時点でスクリプトを中断させる。

結果、`main@origin` が存在しない＝**まだ一度も push していない全リポ**で、
初回 push の前にこのゲートが落ちる。latent bug。

## 回避策（hyoui 側で暫定対応済み）

hyoui では `kawaz.semver.checkBumped` の利用をやめ、同名・同セマンティクスの
ローカル `semver:check-version-bumped` task に差し替えた。先頭で
`jj log -r main@origin` の存在を確認し、無ければクリーンに skip（初回 push 扱い）、
有れば `bump-semver` に bump 比較を委譲する。pkf-tasks 側が修正されたら
`(kawaz.semver.checkBumped){...}.check` に戻せるよう Taskfile.pkl にコメントを残してある。

## 修正案（pkf-tasks 側）

`compareRef` が解決できない（リビジョン不在）ケースを正常系として扱う:

- 案 A: `jj diff` を複合コマンドの外に出し、`compareRef` の存在チェックを先に
  単独で行い、不在なら「triggerPaths に変更なし扱い（＝ bump 不要）」で早期 return。
- 案 B: command substitution の中で `set -e` の影響を切る（`diff_out=$(... || true)`
  等）が、本当のエラーを握り潰すので非推奨。
- 案 A 推奨。初回 push（compareRef 不在）は「比較対象が無い＝ gate を skip」が
  意味的にも正しい。

## 関連

- kawaz/hyoui の `Taskfile.pkl`（ローカル `semver:check-version-bumped` と回避理由コメント）
- kawaz docs-structure.md（canonical Taskfile.pkl テンプレ、`kawaz.semver.checkBumped` の説明）
