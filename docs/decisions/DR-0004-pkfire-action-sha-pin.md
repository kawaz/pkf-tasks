# DR-0004: GitHub Actions の composite action 参照は commit SHA で pin する

- Status: Superseded by upstream fix (mizchi/pkfire 0.5.0+ ships `v<version>` action tags)
- Date: 2026-05-11
- Superseded: 2026-05-12

> **2026-05-12 update**: mizchi/pkfire 0.5.0 で **action 用 `v<version>` 系の tag** (`v0.6.0` / `v0` 等) が別途切られた。`uses: mizchi/pkfire@v0.6.0` で `@` in ref name の workflow parser 問題が起きずに動く (`v0` で major floating も可能)。本 DR の SHA pin 戦略は **不要になった**。pkf-tasks v0.0.14 から `mizchi/pkfire@v0.6.0` に切り替え済。SHA 対応表は historical 記録として残す。

## Context

`mizchi/pkfire` は composite action を `action.yml` 形式で同梱しており、利用側 workflow から `uses: mizchi/pkfire@pkfire@0.4.0` の形で setup できる。pkfire README にもこの記法が推奨例として記載されている。pkfire の release tag 命名は `pkfire@<version>` 形式 (リポ内に複数 package を持つ Pkl 流儀の踏襲)。

しかし `kawaz/pkf-tasks` の CI / release workflow でこの記法をそのまま使うと、GitHub Actions が **workflow file レベルで parse 失敗** し、`run` の name が `.github/workflows/<file>.yml` のまま表示され、jobs が 1 件も評価されない状態になる (失敗 run は log すら取れない、`This run likely failed because of a workflow file issue` のみ)。

切り分けの結果:

| 試した記法 | 結果 |
|---|---|
| `uses: actions/checkout@v4` のみ | ✅ success (parse 成功、`name: CI` 反映) |
| `uses: mizchi/pkfire@pkfire@0.4.0` (README 推奨記法) を追加 | ❌ failure (workflow file issue) |
| `uses: mizchi/pkfire@0.4.0` (`pkfire@` 抜きで `@` 1 個に) | ❌ failure (同様の workflow file issue) |
| `uses: mizchi/pkfire@6e8f4735acd74407455ee6be3debfe9ca8c8e4e9` (commit SHA) | ✅ success |

**真の原因は「`@@` 2 重」ではない**: `@` 1 個版 (`mizchi/pkfire@0.4.0`) でも同じ failure になる。tag 名 `0.4.0` が mizchi/pkfire リポに存在しない (実 tag は `pkfire@0.4.0` のみ) ことも一因だが、それ以前に **`uses: <repo>@<ref>` 構文の `<ref>` 部分が `@` を含む文字列をうまく解決できない** のが本質。GitHub Actions parser が ref 抽出で「最初の `@` で path/ref を区切る」前提に作られている疑いが強く、`pkfire@0.4.0` のような `@` 入り ref を渡すと file-level error で死ぬ。

## Decision

`mizchi/pkfire` を含め、`<repo>@<tag>` 形式の tag を持つ composite action は **常に commit SHA で pin** する。tag-based pin は GitHub Actions parser の制約を引きやすい。

```yaml
# CI / release workflow
- name: Setup pkfire (pkfire@0.4.0)
  uses: mizchi/pkfire@6e8f4735acd74407455ee6be3debfe9ca8c8e4e9
```

SHA pin は副次効果として security best practice にも合致する (上流が tag を移動しても影響を受けない、supply chain attack の耐性)。

## SHA 取得方法

```bash
curl -sL 'https://api.github.com/repos/mizchi/pkfire/git/refs/tags/pkfire@<version>' \
  | grep '"sha"' | head -1
```

pkfire の各 version と SHA の対応:

| pkfire version | commit SHA |
|---|---|
| 0.4.0 | 6e8f4735acd74407455ee6be3debfe9ca8c8e4e9 |

新版に上げる時は SHA を確認して更新。コメントに pkfire version を併記して、人間が読んで version を識別できるようにする。

## Rationale

### 不採用案

**1. tag 名を URL encode (`pkfire%400.4.0`)**

`uses: mizchi/pkfire@pkfire%400.4.0` を試す。**不採用**: GitHub Actions parser は ref を URL encode で受け付けない可能性が高い (未検証だが、ref name 自体は URL safe な文字列を想定されている)。

**2. composite action を使わず、release tar.gz を curl + tar で展開する inline step**

```yaml
- run: |
    curl -fsSL -o /tmp/pkf.tar.gz https://github.com/mizchi/pkfire/releases/download/pkfire@0.4.0/pkf-linux-amd64.tar.gz
    tar xzf /tmp/pkf.tar.gz -C /usr/local/bin
    # ... pkl も同様にインストール
```

**不採用 (現時点)**: composite action のロジック (OS/arch 自動判定、pkl のインストール、PATH 追加) を自前で書き直すコスト。SHA pin で済むなら upstream を使うほうが保守的。

**3. fork して `v<version>` tag を切り直す**

`kawaz/pkfire-fork` のような fork を作って `v0.4.0` tag を切る。**不採用**: upstream への追従コストが膨大、ライセンス的にも `mizchi/pkfire` の改変扱いになる。upstream に「`v<version>` tag も切る or 別アクション repo を切り出す」を要望する選択肢はある (issue 候補)。

### 上流への要望候補 (将来検討)

`mizchi/pkfire` 本体の README で `uses:` 推奨が `pkfire@<version>` 形式のままだと、新規利用者が同じ罠を踏む可能性がある。issue を立てて以下の選択肢を提示する余地:

1. action を別 repo (`mizchi/pkfire-action`) に切り出して `v<version>` 形式の tag で配布
2. README に「実運用では SHA pin 推奨、tag pin だと GHA parser が壊れることがある」を注記

ただし pkf-tasks 側の運用としては SHA pin で完全に解決できているので、急ぎではない。

## Consequences

- workflow 内の `uses:` は SHA 文字列で readability が落ちる → コメントで version を明記してカバー
- pkfire の新版に追従する時は CHANGELOG.md / 本 DR の対応表 / workflow の `uses:` SHA を **3 箇所更新する** 必要。renovate / dependabot の `gomod` 系 manager だと SHA bump を自動化できる (未導入、必要になれば検討)
- 副次効果として supply chain security の観点で SHA pin は推奨パターン

## 関連

- 実装: `.github/workflows/ci.yml` / `.github/workflows/release.yml` の `uses: mizchi/pkfire@<sha>`
- 切り分け過程: kawaz/pkf-tasks の初回 CI 設定 commit と debug 系 commit (run 25654196544 〜 25654395791)
- 上流: `mizchi/pkfire` の `action.yml` (composite action 定義)、README 推奨記法 `uses: mizchi/pkfire@pkfire@<version>`
