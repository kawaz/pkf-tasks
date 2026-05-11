# DR-0002: VCS 検出を Pkl 評価時ではなく cmd 内実行時に行う

- Status: Active
- Date: 2026-05-11

## Context

`tasks/vcs/auto.pkl` は「jj / git のいずれかを環境に応じて選ぶ entry point」として設計した。最初は Pkl の `read?("file:.jj/repo")` で評価時に `.jj` の存在を判定して `jj` or `git` モジュールに分岐する案を試みた。

しかし Pkl の `read("file:...")` は **絶対パスを要求** する仕様で、相対パスでの file resource read は構文エラーになる。さらに、Pkl の評価は purity を保つために CWD-relative な FS state を意図的に見えなくしている (これにより評価結果が deterministic に保たれる — 同じ Pkl コードはどこから evaluate しても同じ output を返す)。

ライブラリ側で「利用者の CWD の `.jj` / `.git` を見て分岐」したいが、Pkl 評価時には不可能。

## Decision

VCS 検出を **Task の cmd 内 bash で実行時に行う**。`auto.pkl` は jj 実装と git 実装の両方を import し、各 task の cmd を bash の `if jj root >/dev/null 2>&1; then ... elif git rev-parse --git-dir >/dev/null 2>&1; then ... fi` 構造で連結した文字列に組み立てる。

```pkl
// auto.pkl
extends "iface.pkl"
import "jj.pkl" as jj
import "git.pkl" as git

local function autoCmd(jjCmd: String, gitCmd: String): String = """
  if jj root >/dev/null 2>&1; then
  \(indent(jjCmd))
  elif git rev-parse --git-dir >/dev/null 2>&1; then
  \(indent(gitCmd))
  else
    echo "vcs: no jj or git repository found from working directory" >&2
    exit 1
  fi
  """

commit = (jj.commit) {
  description = "vcs commit (auto-detect: jj describe+new or git commit)"
  cmd = autoCmd(jj.commit.cmd, git.commit.cmd)
}
// ... 同様に push / fetch / ensureClean
```

`(jj.commit) { cmd = ...; description = ... }` という Pkl の object-amends 構文で **jj 版を base に cmd と description だけ上書き** する。これにより `params` / `env` / `cache` / `shell` 等の他フィールドは jj 版の値を継承する (jj 版と git 版でこれらのフィールドに差が出る設計にしない限り問題ない)。

検出順序は `.jj` 優先 → `.git`。kawaz の git-bare + jj-workspace 構成では両方並存するが、jj が正というポリシー (`jj` 側の revset 言語は git ref のスーパーセットなので、jj 経由で操作できれば git 経由でも動く)。

## Rationale

### 不採用案

**1. Pkl 評価時に `read?("file:.jj")` で分岐する**

最も自然に見えるアプローチ。**不採用**: Pkl の `read("file:...")` は絶対パスのみ受け付け、相対パスは構文エラーになる。`read("file:.jj")` を試すと `File URIs must have a path that starts with '/' (e.g. file:/path/to/my_module.pkl). To resolve relative paths, remove the scheme prefix (remove "file:")` というエラー。`read("file:/Users/.../bump-semver/main/.jj")` のような絶対パスは利用側でしか書けない (ライブラリ側で書きようがない)。

**2. 環境変数 (`KAWAZ_VCS=jj` 等) で分岐する**

direnv で `export KAWAZ_VCS=jj` しておく案。`read("env:KAWAZ_VCS")` は動く。**不採用**: 利用者の dotfile に依存する形になり、ライブラリの transparent な動作にならない。「pkf-tasks をインストールしたら direnv 設定も必須」は学習コストが高すぎる。

**3. Pkl 評価時に jj / git 両方の cmd を持つ Task を生成しておいて、利用側で選ぶ**

`tasks { vcs.commit_jj; vcs.commit_git; ... }` のように全 task を 2 倍生やしておく案。**不採用**: 利用側の `pkf list` が 4 → 8 task に膨らんで読みにくい。利用側で「どちらを使うか」を毎回宣言しないといけないのも逆効果。

**4. 利用側で `import "...jj.pkl"` か `import "...git.pkl"` のどちらかを書く**

`auto.pkl` を提供せず、利用者が自分で選ぶ案。**不採用**: VCS が自動切替されるのが本ライブラリの売り。利用者に選ばせるなら本ライブラリの存在意義が薄い。

### 設計上のポイント

#### `jj root` / `git rev-parse --git-dir` を使う理由

旧版 (v0.0.2 まで) は `[ -d .jj ]` / `[ -d .git ]` で判定していたが、`jj workspace add` で作ったサブディレクトリ等 cwd 直下に `.jj` / `.git` が無いケースで誤検出する罠を踏んでいた。`jj root` / `git rev-parse --git-dir` は親方向に repository ルートを **upward search** してくれるので、workspace のどこから起動しても正しく判定できる。

#### `(jj.commit) { cmd = ... }` で jj 版を base にする bias

jj 版を base にするのは恣意的だが、kawaz 環境では jj が default 想定。git 専用環境からは「`description` が jj 用語を踏襲している」「`params` の説明文が jj 寄り」程度の見栄えの違いしか無い (cmd は両方の if-else を持つので動作は正しい)。完全 vendor-neutral にしたければ iface.pkl 側で description などの共通文言を宣言する案もあり、将来必要なら別 DR で扱う。

## Consequences

- **Pkl 評価結果はキャッシュ可能** — Pkl の deterministic な評価が維持される。同じ pkf-tasks zip から評価される Task の cmd は環境に関わらず同一文字列になる (`if ...; then jj ...; elif ...; then git ...; fi` 形式)
- **実行時のオーバーヘッド** — 各 task 実行のたびに `jj root` / `git rev-parse` を呼ぶ。実測 < 10ms 程度なので実害なし
- **エラーメッセージ** — 両方失敗時は `vcs: no jj or git repository found from working directory` を stderr に出す。利用者が「ここは VCS 配下じゃない」と気付ける文言
- **将来 svn / fossil 等を追加する場合** — `autoCmd` の if-else に分岐を増やすだけで対応可能。iface.pkl の abstract プロパティを増やす場合は実装側 (jj.pkl / git.pkl) もそれぞれ実装する必要があり、これは Pkl の型システムで強制される

## 関連

- 実装: `tasks/vcs/auto.pkl` (`autoCmd` / `indent` 関数)
- 検出ロジックの変遷: `0.0.2` (`[ -d .jj ]`) → `0.0.3` (`jj root` / `git rev-parse`、本 DR の方針確定)
- 上位ドキュメント: `README.md` / `README-ja.md` Design セクション、`docs/DESIGN-ja.md` (今後追加予定)
