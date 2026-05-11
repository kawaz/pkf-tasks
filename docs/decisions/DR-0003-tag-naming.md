# DR-0003: tag 命名は `pkf-tasks@<version>` (pkfire 流儀)

- Status: Active
- Date: 2026-05-11

## Context

新規 Pkl package を GitHub Release 経由で `pkg.pkl-lang.org` proxy で配布するにあたり、tag 命名規約を決める必要がある。候補:

1. **`v<version>`** — Go module / 一般的 SemVer 慣習。例: `v0.0.5`
2. **`<version>`** — SemVer そのまま。例: `0.0.5`
3. **`<name>@<version>`** — pkfire / Pkl monorepo の慣習。例: `pkf-tasks@0.0.5`

Pkl の `packageZipUrl` は `https://github.com/<org>/<repo>/releases/download/<tag>/<name>@<version>.zip` を想定しており、`<tag>` は `<name>@<version>` 形式を **強く期待** している (pkfire 自身もこの形式)。

## Decision

tag 命名は `pkf-tasks@<version>` 形式とする。例: `pkf-tasks@0.0.5`。

`PklProject` の `packageZipUrl` / `sourceCodeUrlScheme` も同じ形式で組み立てる:

```pkl
packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
sourceCode = "https://github.com/kawaz/pkf-tasks/tree/\(name)@\(version)/tasks"
```

GitHub Release を作る際は `gh release create 'pkf-tasks@0.0.5' ...` のように tag を直接指定。release.yml ワークフローも `on: push: tags: ['pkf-tasks@*']` で trigger する。

## Rationale

### 不採用案

**1. `v<version>`**

Go module / Cargo / git-flow の慣習。**不採用**: Pkl package metadata の `packageZipUrl` を `<name>@<version>.zip` 形式から外す必要が出る。pkfire 流儀から外れるため、将来 Pkl ecosystem 全体で何らかの自動化が走った場合に追従できない可能性。

**2. `<version>`**

シンプル。**不採用**: 同じ repo に複数 package を同居させたくなった時 (例: `kawaz/pkf-tasks` の中に `vcs` / `docs` / `go` を別 package として publish する将来案) に衝突する。`<name>@<version>` 形式は将来の monorepo 化への余地を残せる。

### 設計上のポイント

#### `@` を含む tag は GitHub / git のツールチェイン的に問題ないか

- `git tag pkf-tasks@0.0.5` は git の tag 命名規則 (`refs/tags/...` の path 制限) を満たす。`@` は path 上の正規文字
- GitHub の `gh release create 'pkf-tasks@0.0.5'` は問題なく動く (実績あり)
- GitHub release の URL は `pkf-tasks%400.0.5` のように `@` が `%40` に URL encode される。Pkl client は両形式の 302 redirect を正しく追跡する (curl で実測)
- ただし Go module の semver 検出 (`v<version>` 期待) や goreleaser / 一部 release automation tools は **本形式の tag を semver tag として認識しない**。pkf-tasks は Pkl package で Go module ではないため実害なし

#### 互換性のある SemVer pinning

利用側は `package://...kawaz/pkf-tasks/pkf-tasks@0.0.5` のように `pkf-tasks` の name + `@version` で参照。Pkl の package URI 仕様で `<name>@<major>` 形式の解決は **0.x.y では major 単独 pin は不可** (`0.x` 系は major そのものを破壊的変更扱いする SemVer 規約)、利用側は exact version pin が必要。

`0.1.0` 以降は `pkf-tasks@1` のような major-only resolve が Pkl 側で可能になる予定 (Pkl の package resolver 仕様)。

## Consequences

- README の利用例で `pkf-tasks@<version>` を直接書く。利用者は GitHub Release ページで最新版を確認して exact pin する運用 (CHANGELOG.md でも明示)
- 0.0.x 段階では **每 release ごとに利用側 PklProject の dep version を bump する必要**。`pkl project resolve` がエラーを出すので気付けるが、自動化はない (dependabot 等は未対応)
- 0.1.0 以降は `pkf-tasks@0` のような major pin で minor 以下を自動追従可能になる (予定)。CHANGELOG.md にその旨を移行ガイドとして書く

## 関連

- 実装: `tasks/PklProject` (`packageZipUrl` / `sourceCodeUrlScheme`)
- CI 連携: `.github/workflows/release.yml` (`on: push: tags: ['pkf-tasks@*']`)
- 配布: GitHub Releases (`gh release list --repo kawaz/pkf-tasks`)
- 先行事例: `mizchi/pkfire` の `pkfire@<version>` 命名
