# 2026-05-13 — pkg.pkl-lang.org proxy の URI 解決ロジック実証

## 動機

`package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.0#/all.pkl` のような URI で、`pkf-tasks` が 2 回出てくる冗長性を短縮できないか? 加えて SemVer の pre-release / build-metadata identifier を proxy がどう扱うかを実証する。

3 種の experimental release (`pkf-tasks@2.2.1-rc.uri-test` / `pkf-tasks@2.2.1` / `pkf-tasks@2.2.1+experiment`) を作って、各 URI 形式での解決を試した。

## 結果 matrix

### version 形式の解決可否

| version 形式 | tag 作成 | release.yml 発火 | 標準 asset 生成 | pkg.pkl-lang.org 解決 |
|---|---|---|---|---|
| `2.2.0` (core SemVer) | ✓ | ✓ | ✓ | **✓** |
| `2.2.1-rc.uri-test` (pre-release) | ✓ | ✓ | ✓ | **✗** (404) |
| `2.2.1` (core SemVer) | ✓ | ✓ | ✓ | **✓** |
| `2.2.1+experiment` (build-metadata) | ✓ | ✓ | ✓ | **✗** (404) |

GitHub release / release.yml / asset 生成は **どの形式でも問題なく通る**。ただし pkg.pkl-lang.org proxy は **core SemVer (`X.Y.Z`) のみ** 解決対象とする。pre-release `-` / build-metadata `+` を含む version は全て 404。

`%2B` で URL encode しても結果は同じ (pkl 側で `hasVersion` 違反、proxy も 404)。`+` literal も `%2B` encoded も両方 NG。

### URI 構造の解決可否 (`pkf-tasks@2.2.1` での検証)

| URI 形式 | 構造 | 解決可否 | 404 元 |
|---|---|---|---|
| `.../kawaz/pkf-tasks/pkf-tasks@2.2.1` | baseline (4-segment、`<name>` = `<repo>`) | ✓ | — |
| `.../kawaz/pkf-tasks/@2.2.1` | 4-segment、`<name>` 空 | ✗ | proxy (pkg.pkl-lang.org) |
| `.../kawaz/pkf-tasks@2.2.1` | 3-segment、name 抜き | ✗ | proxy (pkg.pkl-lang.org) |
| `.../kawaz/pkf-tasks/pkf@2.2.1` | 4-segment、`<name>` ≠ `<repo>` (短縮 name) | (✓ proxy parse 通過、✗ GH 404 — asset 不在のため) | github.com |

短縮 name のエラーメッセージが決定打:

```
404 when making request `GET https://github.com/kawaz/pkf-tasks/releases/download/pkf@2.2.1/pkf@2.2.1`.
```

つまり proxy は:

1. URI を `<host>/<owner>/<repo>/<name>@<version>` の **4-segment 強制 parse**
2. `<repo>` と `<name>` は **別物として扱う** (URI path 上の位置で区別)
3. `<name>` 部分を **tag 名 + asset 名の prefix として GitHub に問い合わせ**
   (`releases/download/<name>@<version>/<name>@<version>` を見に行く)
4. 「name 空」「name 抜き」は proxy 自身で 404 (= GitHub 問い合わせ前に reject)
5. 「name = `<repo>` と別文字列」は proxy parse は通る、release tag/asset が一致してれば解決可能

## proxy ロジック推定

```
package://pkg.pkl-lang.org/<host>/<owner>/<repo>/<name>@<version>
                            └──────path part (4-segment 必須)────┘

proxy が GitHub に問い合わせる URL:
  https://github.com/<owner>/<repo>/releases/download/<name>@<version>/<name>@<version>
                                                       └─ tag ─┘  └─ asset ─┘
```

- `<owner>` / `<repo>`: URI path から取得 (位置固定)
- `<name>@<version>`: URI path 末尾 segment、これをそのまま tag 名 + asset 名 prefix に使う
- core SemVer (`X.Y.Z`) のみ受け入れ、pre-release / build-metadata は reject

## URI 短縮の余地

実用上の唯一の自由度: **`<name>` を `<repo>` と違う短い文字列にする**。

- 現状: `.../kawaz/pkf-tasks/pkf-tasks@2.2.0` (= 24 文字 path suffix)
- 短縮: `.../kawaz/pkf-tasks/pkf@2.2.0` (= 19 文字、5 文字短縮)

ただし採用するには:
- PklProject の `name = "pkf"` に変更
- release tag を `pkf@<version>` に変更
- release asset 名も `pkf@<version>` に
- release.yml の verify step 修正

**メリット (5 文字短縮) < デメリット (命名混乱、リポ名と package 名が一致しない違和感)** で実用採用は推さない。kawaz/pkf-tasks では現状の `<name>` = `<repo>` を維持する。

## pre-release / build-metadata の扱い

pkfire の migrate task の regex は SemVer 2.0.0 完全対応 (`-[0-9A-Za-z.-]+` / `+[0-9A-Za-z.-]+` を含む) で **string parse は通る** が、**実際の resolver / proxy は core SemVer のみサポート**。

つまり利用側で pre-release / build-metadata version を tag/PklProject に使うと:

- GitHub release は作成できる (release.yml も正常完走)
- 他リポからこの release を Pkl package として **import できない** (404)

→ kawaz/* リポでは **pre-release / build-metadata の tag/version は使わない**。stable release (`X.Y.Z`) のみ。

## 実用上の指針

1. **URI 形式は現状の `<host>/<owner>/<repo>/<name>@<version>` を維持** (短縮余地は実用メリット薄い)
2. **`name` は `<repo>` と一致** を慣習として維持 (混乱回避)
3. **pre-release / build-metadata の release tag は禁止** (proxy で解決不能、利用者が hit する 404 はバグに見える)
4. release.yml の tag glob (`pkf-tasks@*`) は `+` を含むタグでも発火するが、利用者から見て使えない release を量産しても意味がない

## 4 asset 構造の解明

`curl https://pkg.pkl-lang.org/.../pkf-tasks@2.2.0` で proxy を直接叩いた結果、proxy は **単純な 301 redirect** で動作:

```
GET https://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.0
↓ 301 Location:
https://github.com/kawaz/pkf-tasks/releases/download/pkf-tasks@2.2.0/pkf-tasks@2.2.0
                                                      └─ tag 名 ─┘ └─ asset 名 ─┘
```

= URI 末尾 segment `<name>@<version>` を **tag 名 + 拡張子なし asset 名** の両方として使う (両者は同じ文字列で固定)。

### 拡張子なし asset の中身 (= metadata JSON)

```json
{
  "name": "pkf-tasks",
  "packageUri": "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@2.2.0",
  "version": "2.2.0",
  "packageZipUrl": "https://github.com/kawaz/pkf-tasks/releases/download/pkf-tasks@2.2.0/pkf-tasks@2.2.0.zip",
  "packageZipChecksums": { "sha256": "394ec62f...dd514a62" },
  "dependencies": { "pkfire": { "uri": "...", "checksums": { "sha256": "..." } } },
  "sourceCodeUrlScheme": "https://github.com/kawaz/pkf-tasks/blob/pkf-tasks@2.2.0/tasks%{path}#L%{line}-%{endLine}",
  "sourceCode": "https://github.com/kawaz/pkf-tasks/tree/pkf-tasks@2.2.0/tasks",
  "license": "MIT",
  "authors": ["..."]
}
```

### release.yml が生成する 4 asset

| asset 名 | 種類 | 役割 |
|---|---|---|
| `<name>@<ver>` (拡張子なし) | metadata JSON | proxy redirect 先、`packageZipUrl` / `packageZipChecksums` 等を記述 |
| `<name>@<ver>.sha256` | テキスト | metadata 自体の SHA256 (verify 用、resolver が直接使うかは未検証) |
| `<name>@<ver>.zip` | zip | 実 package 内容 (Pkl files)、metadata の `packageZipUrl` から間接 fetch |
| `<name>@<ver>.zip.sha256` | テキスト | zip の SHA256 (ただし metadata JSON 内 `packageZipChecksums.sha256` でも同じ hash あり、独立ファイルは crutch かも) |

### 最低何ファイル必要か

理屈上は **metadata + zip の 2 ファイル**で動く可能性 (metadata 内に zip checksum が JSON で書かれてるため)。ただし実証は未実施。実用上は release.yml が 4 ファイル全部生成するので「足りない問題」は起きない。

### URI 末尾 segment の `<name>@<version>` 制約 (まとめ)

- `<name>` 必須 (空文字 / 抜き ✗)
- `<name>` ≠ `<repo>` でも proxy parse は通る (= `<name>` を別名にする自由度あり、実用メリット薄い)
- `<version>` は **core SemVer (`X.Y.Z`) のみ**
- pre-release `-` / build-metadata `+` は proxy で 404

### 個人的感想

「拡張子なし `<name>@<version>` 拡張子なし asset」という強烈な違和感を最初に受け入れた時点で、`.sha256` 2 つ追加 / 4 ファイル運用は気にならなくなる (= **ドアインザフェイス的な仕様体験**)。慣れる前提で運用すれば良い。

## 関連

- `docs/journal/2026-05-12-pkl-package-uri-no-v-prefix.md` — `v` prefix の制約 (`hasVersion` 違反)
- DR-0003 (tag 命名) — `<name>@<version>` 一系統採用の根拠
- 検証日時: 2026-05-13
- 検証時 pkl 0.31.1 / pkfire 0.7.0 / pkg.pkl-lang.org 公式 proxy

## 実験 release (cleanup 対象)

以下の experimental release は本実証のためのもので、利用者向けではない。retain か削除かは後日判断:

- `pkf-tasks@2.2.1-rc.uri-test` — pre-release、proxy 解決不能を実証
- `pkf-tasks@2.2.1` — 本 patch、URI 構造 4 パターンを実証
- `pkf-tasks@2.2.1+experiment` — build-metadata、proxy 解決不能を実証

`2.2.1` を消すと将来の本 patch 2.2.1 は **`2.2.2` から再開**になる。
