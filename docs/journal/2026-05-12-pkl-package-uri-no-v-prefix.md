# 2026-05-12 — Pkl package URI に `v` prefix は付けられない

## 結論

Pkl package URI は **SemVer 厳守 (`v` prefix 不可)**。`pkfire@v0.6.0` のような形式は Pkl resolver が拒否する。GH Actions の `uses: <repo>@v<ver>` 慣習とは別系統。

## 検証

```
$ cat /tmp/pkl-v-test/PklProject
amends "pkl:Project"
dependencies {
  ["pkfire"] {
    uri = "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@v0.6.0"
  }
}

$ pkl project resolve
–– Pkl Error ––
Type constraint `hasVersion` violated.
Value: "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@v0.6.0"

234 | typealias PackageUri = Uri(startsWith("package:"), hasVersion)
                                                         ^^^^^^^^^^
at pkl.Project#RemoteDependency.uri (https://github.com/apple/pkl/blob/0.31.1/stdlib/Project.pkl#L234)
```

`hasVersion` 制約 (Pkl stdlib `Project.pkl` の `PackageUri` typealias) が `v` prefix を拒否。SemVer 2.0.0 spec の version は **数字のみ** (`v` は version 文字列の外、tag 慣習で別) なので、Pkl の挙動は仕様準拠。

## エコシステム比較

| エコシステム | tag / URI 形式 | v prefix |
|---|---|---|
| **Pkl package** (`package://...`) | `<name>@<version>` (`pkfire@0.6.0`, `pkfire@1`) | × |
| **GitHub Actions** (`uses: <repo>@<ref>`) | `v<version>` (`v0.6.0`, `v1`) | ○ |
| **Go modules** | `v<version>` (Go の独自仕様、SemVer prefix 強制) | ○ |
| **Cargo / npm** | `<version>` (`1.0.0`) | × |
| **Git tag 慣習一般** | `v<version>` または `<version>` (両方あり) | △ |

## kawaz/pkf-tasks の場合

- pkf-tasks は **Pkl package only** (GH Actions として使われない)
- → tag は `pkf-tasks@<version>` 一系統で十分 (`v` 付き tag を切る必要なし)
- mizchi/pkfire は **両用途** (Pkl + Actions composite action) なので 2 系統 (`pkfire@<ver>` / `v<ver>`) を `v-tags.yml` で同期生成

## 利用者が混乱しないように

利用者が GH Actions 慣習から「`@v1` でないと駄目?」と疑問を持つ可能性はあるが、これは README に書くより:

- Pkl resolver が `hasVersion` で **明確なエラーメッセージ** を出す (上記)
- pkl-lang.org docs に SemVer 規定がある

ので、混乱した利用者は自然と解にたどり着く。`README.md` は「使い方」を示す場であって、Pkl 仕様の説明は範囲外。本 journal で経緯を残せば十分。

## 関連

- DR-0003 (pkf-tasks の tag 命名は `<name>@<version>` 形式) — pkfire 流儀踏襲、v なし
- DR-0004 (GHA composite action SHA pin、その後 mizchi 0.5.0+ の `v<ver>` tag で解決、Superseded) — Actions 側は v 付きを使う対比例
- Pkl stdlib: https://github.com/apple/pkl/blob/0.31.1/stdlib/Project.pkl
- SemVer 2.0.0 spec: https://semver.org/
