# DR-0009: push 後フローを followup group として設計

- Status: Active
- Date: 2026-05-18

## Context

`pkf run push` で作業が完結するリポは意外と少ない。push 後に行うべき処理が、リポの性質に応じてパターンとして存在する:

- **CI workflow の成否確認** — push したら CI が回る。その成否を見届ける (ほぼ全リポ共通)
- **release 後のローカル反映** — Homebrew 配布なら `brew update && brew upgrade`、Claude plugin なら `plugin marketplace update` + ユーザへ `/reload-plugins` 案内
- **デプロイ後の検証** — 本番 URL での E2E テスト 等

git には `post-push` hook が無く、push 後の処理をフックで強制する手段がない。一方、タスクランナー + AI 運用なら「push は完結させ、次アクションを示唆 → AI/人間が後続を実行」が素直。これらの後続処理を、強制ではなく **自然に進むフロー** として設計したい。

## Decision

### 1. push と followup を独立した 2 task に分離

- `push`: ゲート (ci 等) + 送信。ここで**完結**。exit code = push の成否。
- `followup`: push 後の後続処理を担う**別 task**。
- `push` の deps に followup を入れない (deps は「前提 = 先に実行」で**向きが逆**)。`followup` の deps にも push を入れない (疎結合に保ち「push だけしたい」を自然に許す)。両者は push 完了時の hint で繋ぐ。

### 2. followup は決定論的な pkf task

CI watch (run_id が取れるまでの poll → `gh run watch` → 成否判定) も含め、followup の処理は followup task 内に**決定論的に**書く。「push 完了時に自然言語の hint を出力し、AI がそれを解釈して Monitor 等で実行する」案は不採用 (Rationale 参照)。

### 3. followup 内部の順序は chain deps で表現

pkfire の deps は並列実行され、sibling 間は**アルファベット順**で、宣言順は保持されない。順序保証の手段は **chain deps** (A を B の deps にする一本道) のみ。

followup は `followup:ci-watch → followup:<反映> → … → followup` (umbrella) の chain として組む。

- 副産物: deps が失敗すると後続 task はスキップされる pkfire 標準挙動により、「CI 失敗時に後続を止める」が自動で得られる。
- `pkf graph --format tree --target followup` で実行順が縦一列に可視化できる。

### 4. `kawaz.followup` group を新設

DR-0007 の group 構造規約に従う。`tasks/followup.pkl` (entry shim) + `tasks/followup/` 配下に実装。task 名は `followup:<action>`。各リポは building block (`followup:ci-watch` 等) を chain deps で組んで自リポの `followup` task を定義する。

group 名の候補として `postpush` / `ship` を検討したが `followup` を採用 (不採用案参照)。

### 5. 担い手で実行経路を分岐させない

- 人間: `pkf run followup` を foreground で実行し、待つ。
- AI: `pkf run followup` を Bash ツールの `run_in_background` で実行 (10分制限は foreground のみ、detached なら回避) し、完了通知を待つ。**Monitor は不要**。
- 中身は同一 task。担い手が違っても実行されるものは同じ。

### 6. 人間向けメッセージは stdout 経由

「ユーザに `/reload-plugins` を伝える」等、人間にしか向けられない指示は followup task の stdout に種別マーク付き (例: `📢`) で出力する。AI はそれを読んで user に転送する。task 自身は人間に語りかけられないので stdout を介する。

### 7. action カタログと chain 構成

followup chain は**監視対象（責務の所在）**で 4 層に分ける:

```
followup:ci-watch → followup:registry-watch-<reg> → followup:registry-apply-<reg> → followup:e2e
  自リポ + GitHub      外部レジストリへの伝播            ローカル環境への反映              配布物の動作確認
```

- `ci-watch` / `e2e` は registry 非依存。`e2e` は「PATH 経由で `<tool> --version`」程度の**最小デフォルト**を持ち、リポが cmd を上書きできる（中身空のプレースホルダにはしない）。
- `registry-watch` / `registry-apply` は外部パッケージレジストリ（npm / homebrew-tap / mooncake 等）ごとに実装が異なる。`tasks/followup/registry/` 配下に iface (`abstract watch` / `abstract apply`) + レジストリごとの実装 module を置く。vcs group の iface + jj/git 実装分離と**同型**（DR-0001 / DR-0007 準拠）。auto-dispatch は不要（リポが配布先を明示選択する）。
- レジストリ種別は**有限・離散で判定も一意**（npm は `npm view`、homebrew-tap は formula version 等）。種別ごとの module 分離は handler 乱立ではなく、有限集合への 1:1 実装。「種別を引数で受けテーブル引きする 1 action」案は不採用 — 種別ごとの知識が 1 module に閉じる方が DRY で、vcs group と流儀も揃う。
- 命名は DR-0007 の対称命名規約に従い対象明示 suffix を付ける: `followup:registry-watch-<reg>` / `followup:registry-apply-<reg>`。prefix `followup:registry-` で `pkf list` 上にまとまる。
- レジストリ実装は全種を最初に網羅せず、利用リポが実際に使うものから**順次追加**する（YAGNI）。iface だけ先に固定すれば実装追加は局所的で済む。

## Rationale

### なぜ決定論的 task か

「AI が自然言語 hint を解釈して実行」案はブレる:

- **解釈ゆらぎ** — 「listing を上から順に」「失敗で停止」を AI が毎回正しくやる保証がない。
- **担い手で経路が分岐** — 人間は `pkf run` で待てるが、AI は Bash の 10分制限があり Monitor 等の別経路が要る。同じ処理なのに担い手で実行方法が変わる。

決定論的 task にすれば AI の役割は「`run_in_background` で起動する」「stdout の指定行を user へ転送する」の 2 点に縮み、ブレが最小化される。

### なぜ push と分離か

followup を push に内包すると:

- push 成功・CI 失敗を**単一 exit code で表せない** (混線)。
- CI watch の長時間ブロッキングを push が抱える。
- feature ブランチへの push にも不要な後続が走る。

独立 task にすれば push の exit code の意味が清潔に保たれ、「push だけしたい」も自然に成立する。

### なぜ chain deps か

cmd 直列スクリプトでも順序は作れるが、chain deps なら各ステップが pkf task として `pkf list` / `pkf graph` に現れ、「CI watch だけ再実行」のような部分操作ができる。pkfire の構造に素直。

## 不採用案

1. **followup を AI 主導ワークフロー化** (自然言語 hint listing + AI 解釈 + Monitor) — 上記 Rationale の通りブレる。
2. **followup を push に内包** (push の後段 / deps) — exit code 混線・ブロッキング・deps の向きが逆。
3. **`ship` という group/task 名** — lib リポには「出荷」概念が無く浮く。「push 後」を表す中立語が必要。E2E もデプロイ確認も「followup の一種」として包含できる。
4. **`postpush` という group 名** — camelCase が既存 group (`vcs`/`docs`/`semver`/`migrate`) と不揃い。全小文字化した `postpush` も子音連続で読みにくく、push 専用に固定され将来の「bump-version 後」「release 後」等に使い回せない。`followup` は全小文字 1 トークンで揃い、push に縛られない。
5. **followup を単一 cmd 直列スクリプト** — chain deps に対し、部分再実行ができず `pkf graph` での可視化も劣る。
6. **CI watch を Monitor ツールで実行** — Bash の `run_in_background` で 10分制限を回避でき完了通知も得られるため、Monitor は過剰。

## Consequences

- pkf-tasks に `kawaz.followup` group を追加する (次 minor または major)。
- canonical Taskfile.pkl テンプレ (`kawaz/bump-semver`) に followup task の組み立て例を追加。`docs-structure.md` ルールのテンプレ節も更新。
- 後続で詰める (本 DR では未確定):
  - 各 action の具体的なコマンド実装 (`ci-watch` の run_id poll ロジック、各レジストリ module の `watch` / `apply` コマンド)。
  - push 完了時の hint 出力の具体形 (cmd 末尾で何を出すか)。
  - `check-version-bumped` を push に残しつつ feature ブランチで自動 skip する `target` 判定 (push task = `kawaz.vcs.push` の API 変更) — 別 DR 候補。

## 関連

- DR-0007: group 構造規約 — followup group の追加はこれに従う
- DR-0005: semver group 新設 (check-bumped) — `target` 判定の後続 DR がこれに関連する
