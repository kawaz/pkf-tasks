# pkf-tasks

> [English](./README.md) | 日本語

[mizchi/pkfire](https://github.com/mizchi/pkfire) (Pkl で書く typed タスクランナー) 向けの共通タスクモジュール集。kawaz/\* の各リポジトリで再利用される pkfire テンプレを 1 箇所に集約する。

## 設計方針

- **abstract module + 値レベル選択**: `tasks/vcs/iface.pkl` のような抽象モジュールを `tasks/vcs/jj.pkl` / `tasks/vcs/git.pkl` が `extends` で実装 (`abstract module` を `amends` するのは Pkl 制約上不可、`extends` 必須)。`tasks/vcs/auto.pkl` が実行時 cmd 内で `jj root` / `git rev-parse` の有無で実装を切り替える
- **同一インターフェース**: jj/git の実装は同じプロパティ (`commit` / `push` / `fetch` / `ensureClean`) を持つので、利用側は `vcs.push` のように透過的に参照できる
- **package URI 配布**: `package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>` で参照。Pkl のキャッシュ機構 (`~/.pkl/cache/package-2/`) で SHA256 検証 + オフライン対応

## 利用例

各プロジェクトの `Taskfile.pkl` から (バージョンは最新の GitHub Release を参照):

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.4.0#/Taskfile.pkl"

import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.5#/vcs/auto.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.5#/docs/translations.pkl" as docs

local push: Task = new {
  name = "push"
  deps { vcs.ensureClean; docs.checkTranslations }
  cmd = "..."
}

tasks { push; docs.checkTranslations; vcs.push }
```

## モジュール一覧

| Path | 内容 |
|---|---|
| `vcs/iface.pkl` | abstract module (commit / push / fetch / ensureClean) |
| `vcs/jj.pkl`    | jj 実装 (extends iface) |
| `vcs/git.pkl`   | git 実装 (extends iface) |
| `vcs/auto.pkl`  | エントリポイント (環境判定で jj / git 切り替え) |
| `docs/translations.pkl` | 翻訳ペア (README / docs/DESIGN / docs/MANUAL 等) の整合性チェック |

将来の追加候補: `go/` (gofmt / vet / test / build), `rust/` (cargo fmt / clippy / build / test), `release/` (bump-version family + push)。

## 開発

```bash
cd tasks
pkl eval vcs/auto.pkl    # ロード確認
pkl project package      # 公開用 zip + metadata 生成
```

リリースは GitHub Release に zip をアップロード (`pkg.pkl-lang.org` が proxy する)。

## 関連

- 共通テンプレを取り込む側の運用ガイド: `~/.claude/rules/docs-structure.md`
- pkfire 本体: <https://github.com/mizchi/pkfire>
- 利用実例: <https://github.com/kawaz/bump-semver>
