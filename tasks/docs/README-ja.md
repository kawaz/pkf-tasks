# `docs/` — ドキュメント整合性チェック

> [English](./README.md) | 日本語

kawaz `docs-structure.md` ルールに基づく翻訳ペア (`*-ja.md` / `*.md`) の存在 / 相互リンク / commit timestamp 順序を検証する。`push` の deps に組み込むことで「英訳忘れ」「先に英訳を更新して原本 (-ja) を放置」を push 時に検知する想定。

v2.1.0+ で 3 つの task (umbrella + sub-check 2 つ) に分割、CLI 引数 / 近所探索で多言語対応も。

## Public entry vs Internal

- **Public entry**: `tasks/docs.pkl` — 利用者が `import` すべき唯一のファイル。export field 名は 2.x public API contract
- **Internal 実装** (直接 import 禁止、minor release で改名/移動の可能性あり):
  - `translations.pkl` — 3 つの Task 定義 + `forPairs(pairs)` factory。`tasks/docs.pkl` から re-export

## 提供 task

### `docs:check-translations` (umbrella)

- 有用性: ★★★
- 構成: `deps { docs:check-translation-commit-lag; docs:check-translation-links }` — sub-check を並列実行
- 注意: pkfire 0.6.0 の orchestrator は umbrella の CLI args を deps に伝播しない。`-- <glob>` を渡したい時は sub-check を直接呼ぶ

### `docs:check-translation-commit-lag`

- 有用性: ★★★
- 内容: 各正本に対して VCS commit timestamp を翻訳先全部と比較。翻訳先が正本より古ければ失敗 (= 翻訳が正本の最新編集に追従していない)
- 多言語対応: 1 正本に N 翻訳先、各 timestamp を独立に比較

### `docs:check-translation-links`

- 有用性: ★★★
- 内容: 相互リンク文字列を冒頭 5 行に検証:
  - `<base>-ja.md` に `> [English](./<base>.md) | 日本語`
  - `<base>.md` に `> English | [日本語](./<base>-ja.md)`
- 2 言語ペア (1 正本 ↔ 1 翻訳先) のみ。翻訳先が 0 個 or 2+ 個の場合は skip ログ + pass (規約は 2 言語前提のため。汎用化は `docs/issue/2026-05-12-link-pattern-injection.md` 参照)

## 正本指定方法

優先順:

### 1. CLI args (`acceptsArgs = true`)

```bash
# 明示リスト
pkf run docs:check-translation-commit-lag -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# glob (** は bash 4+、bash 3.2 では 1 階層のみ)
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'

# bash 3.2 で再帰 glob したいなら brace expansion で代用
pkf run docs:check-translation-commit-lag -- {,*/,*/*/}*-ja.md
```

### 2. Pkl Listing (via `forPairs(...)`)

```pkl
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.2#/docs.pkl" as docs

local myCheck: Task = docs.forPairs(new Listing<String> {
  "README-ja.md"
  "docs/CONTRIBUTING-ja.md"
})

tasks { myCheck }
```

旧 `task(pairs)` は v2.x の `@Deprecated` alias として残る (v3.0 で削除)。

### 3. Auto-discover (デフォルト)

CLI args なし + `pairs` Listing 空のとき、cwd 配下の `*-ja.md` を全て正本扱い。除外ディレクトリ:

- `.jj/`
- `.git/`
- `*/.out/`
- `*/node_modules/`

## 近所探索 (正本 → 翻訳先)

各正本に対して、同じ basename の他ファイルを翻訳先として近所探索 (正本自身は除外)。

| 正本パターン | 翻訳先 |
|---|---|
| `<base>-ja.md` (kawaz 慣習) | `<base>.md` 単体 (1:1、en) |
| `<base>.md` (en 原本など) | `<base>-??.md` / `<base>-???.md` (2-3 文字 language code) |

- `-ja.md` 正本は **1:1 探索** (en のみ) に固定。`data-layout-ja.md` と `data-layout-history-ja.md` の誤検知を防ぐため
- `*.md` 正本は同様の理由で 2-3 文字 language code suffix (`-ja.md`, `-zh.md`, `-jpn.md`, etc.) に限定

## 設計判断

- **VCS commit timestamp** (stat mtime ではなく): jj の workspace 切替で stat mtime が揺れるため。jj → `jj log -T 'committer.timestamp().format("%s")'`、git → `git log -1 --format=%ct -- <file>`
- **Untracked file は ts=0 にフォールバック**: `git log -1 --format=%ct -- <untracked>` が exit 0 + 空 stdout を返すので、`0` に正規化。`[ "" -lt N ]` の silent exit-2 を避けるため
- **1 task / 1 check (`Listing<Task>` fan-out ではなく)**: ペア単位の cache 粒度が要らない場合は 1 task で十分 (pkfire 流儀)
- **リンクパターン hard-code**: kawaz `docs-structure.md` 規約前提。他プロジェクトは自前 check task を実装して `deps` に並べれば良い。パターン注入は `docs/issue/2026-05-12-link-pattern-injection.md` で追跡

## 関連

- `docs/decisions/` — Decision Records
- `docs-structure.md` (kawaz 個人 rule) — 本 task が検査する翻訳ペア規約
