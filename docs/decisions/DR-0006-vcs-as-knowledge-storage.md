# DR-0006: `vcs/{iface,jj,git,auto}.pkl` を VCS knowledge 集積場として位置付ける

- Status: Active
- Date: 2026-05-11

## Context

`vcs/{iface,jj,git,auto}.pkl` は当初「jj/git の Task を抽象化する dispatch layer」として設計した (DR-0001 + DR-0002)。実装している抽象は `commit` / `push` / `fetch` / `ensureClean` の 4 つ。DR-0005 で Pkl function helper (`diffSummary` / `readAtRef`) も追加し、「Task export + function export」の二層構造に拡張した。

しかし運用していくと、higher-level なレシピ (`semver/check-bumped` 等) を書くたびに「jj だとこうやる / git だとこう」の bash 分岐をレシピ内に埋め込む傾向が出てきた。例:

- 「最新 tag を取りたい」: jj は `jj git fetch` + `jj git import`、git は `git fetch --tags origin`
- 「特定 ref のファイル内容を取りたい」: jj は `jj file show -r <ref> -- <path>`、git は `git show <ref>:<path>` (これは DR-0005 で吸収済)

レシピ側 (semver / docs / lint / future ...) で都度 jj/git 分岐を書くと:

- 同じ knowledge が複数レシピで重複
- jj/git 仕様の細かい差異 (例: `jj git fetch` が tag を取らない問題、`git describe` が jj workspace で動かない問題等) を毎回レシピ作者が再発見する
- レシピが「自身の責務」と「VCS 差異吸収」の両方を抱えて肥大化

## Decision

**`vcs/{iface,jj,git,auto}.pkl` を「VCS dispatch + jj/git knowledge 集積場」として明示的に位置付ける**。jj/git で挙動差異がある操作は Task または Pkl function として vcs/* に追加し、レシピ側は VCS 詳細を意識しない。

具体的には、以下のいずれかの形で knowledge を吸収する:

1. **Task として追加** (実行可能、deps として組み立てる) — `vcs:fetch-tags` 等
2. **Pkl function として追加** (bash 断片を返す、他 Task の cmd 内に埋め込む) — `diffSummary` / `readAtRef` (DR-0005)

選択基準:
- 利用側 task の deps に挟みたい (= 順序 + cache key で扱いたい) → **Task として追加**
- 利用側 task の cmd 内で `$(...)` 展開して使いたい (= 文字列断片として組み立てる) → **function として追加**

## Rationale

### 不採用案

**1. レシピ側に jj/git 分岐を都度書く (現状)**

シンプル、library 拡張不要。**不採用**: 上述の通り重複・肥大化・knowledge 散逸。長期的にコスト累積。

**2. vcs/* を「dispatch のみ」に絞り、knowledge は外部 (利用者ドキュメント) に置く**

vcs/* の責務をミニマムに保つ案。**不採用**: kawaz の他リポでも同じ knowledge を必要とするので、ドキュメントだけでは reproduce 必要 (利用者が毎回手書き)。Pkl module であれば 1 度書けば全リポで再利用できる。

**3. VCS layer を別 package に切り出す (`pkf-vcs` 等)**

vcs/* だけ別 repo / package にする案。**不採用**: 現状 vcs/* は pkf-tasks のコア機能で、他 group (semver / docs) との結合度も高い (semver/check-bumped が vcs.diffSummary を借りる等)。分離するメリットより repo 増加コストが上回る。0.1.0 以降に再評価する余地はある。

### 設計上のポイント

#### 「knowledge 集積場」と「責務肥大化」のトレードオフ

vcs/* に何でも詰め込むと「VCS 関連何でも屋」になり境界が曖昧になるリスクは認識する。ただし 0.0.x 段階では:

- まず溜める (knowledge は明示化されないと再利用されない)
- 必要になったら区分する (例: `vcs/iface.pkl` の abstract task が増えすぎたら `vcs/extras.pkl` 等に分離)
- 早すぎる抽象化は避ける (ユーザ rule `design-thinking.md` 「将来仮定的要件のために今の複雑さを増やすな」)

明確な境界線:

- ✅ vcs/* に置く: 「同じ操作を jj/git で別コマンドにする必要がある」もの
- ❌ vcs/* に置かない: VCS 不問の操作 (e.g. SemVer 比較、ドキュメント検証、format)、または利用者の特定 release flow に固有なもの

#### `extends "iface.pkl"` の拡張時の影響

iface.pkl に新規 abstract task を追加すると、`extends "iface.pkl"` する全実装 (jj.pkl / git.pkl / auto.pkl) で実装漏れが評価時エラーになる (DR-0001 の型システム恩恵)。これは pkf-tasks 内部の compile-time 安全性として歓迎すべき挙動。外部 consumer には影響なし (consumer は iface.pkl を直接 extends しない)。

#### Pkl function helper vs Task の選択基準

DR-0005 で導入した Pkl function は「cmd 内に bash 断片として埋め込む」用途。Task は「deps として組み立てる」用途。どちらを選ぶかは「呼び出し側がどう使いたいか」で決める:

- Task 化のメリット: deps 経由で順序保証 + cache key 連動、`pkf list` で発見性高い
- Task 化のデメリット: 引数渡し不可 (pkfire 仕様、DR-0005 で確認)、shell process が分離される
- function 化のメリット: 引数渡し自由、呼び出し側の bash と同一プロセスで動く
- function 化のデメリット: `pkf list` に出ない、利用側で文字列補間が必要

## Consequences

- **v0.0.9 から**: `vcs:fetch-tags` を実装 (jj は `jj git fetch + jj git import`、git は `git fetch --tags origin`)。`semver:check-bumped` の `needsFetchTags = true` 利用で deps として組み合わせる
- 今後、jj/git で挙動が分かれる操作を発見したら vcs/* に追加する流儀を確立
- iface.pkl の abstract task / function が増えると jj.pkl / git.pkl の実装義務も増える。実装が大きくなったら sub-module 分割 (`vcs/jj/` 配下に分割等) を別 DR で検討
- 「VCS knowledge」を 1 箇所に集めることで、新規レシピ作者は「vcs/* を見れば jj/git 差異は吸収済」と仮定できる。再発する議論を減らす

## 関連

- 上位 DR: DR-0001 (abstract module + extends)、DR-0002 (runtime VCS dispatch)、DR-0005 (Pkl function helper / semver group)
- 実装: 本 DR に合わせて `vcs/iface.pkl` に `abstract fetchTags`、`jj.pkl` / `git.pkl` / `auto.pkl` に実装を追加 (v0.0.9)
- 議論経緯: `docs/journal/2026-05-11-pkf-tasks-v0.0.8-semver-compare-experiment.md` の末尾、bump-semver の `vcs:latest-tag()` が jj 経由で古い tag を返した現象から議論が発展
