# docs:check-translation-links のパターン注入対応

- Status: Open
- Date: 2026-05-12
- Priority: Low (kawaz 用途は現状で十分、汎用化要望が出たら対応)

## 現状

`docs:check-translation-links` (v2.1.0+) は ja ↔ en の相互リンク文字列が **hard-coded**:

- `*-ja.md` の冒頭 5 行に `> [English](./<base>.md) | 日本語`
- `*.md` の冒頭 5 行に `> English | [日本語](./<base>-ja.md)`

これは kawaz の docs-structure.md ルールで決めた書式に直接依存している。

## 改善案

利用側でリンク文字列のパターンを **注入** できると、別プロジェクトでも使い回せる library に進化する。

### 案 A: `forPairs` に link pattern を渡す

```pkl
function linksForPairs(
  pairs: Listing<String>,
  jaLinkPattern: String?,
  enLinkPattern: String?
): Taskfile.Task = ...
```

`null` ならデフォルト (kawaz 慣習)、非 null なら override。文字列内で `${base}` のような placeholder 展開。

### 案 B: module amend パターンで override

```pkl
module com.github.kawaz.pkfTasks.docs.translations

jaLinkPattern: String = "> [English](./${base}.md) | 日本語"
enLinkPattern: String = "> English | [日本語](./${base}-ja.md)"
```

利用側:

```pkl
local myCheck = (kawaz.docs.translations) {
  jaLinkPattern = "> [EN](./${base}.md) | JA"
}.checkTranslationLinks
```

semver/check-bumped と同じ流儀で一貫性 ◎。

### 案 C: 多言語化と統合

3+ 言語対応も同時に解決するなら、リンクパターンを **言語ごとに** 定義する仕組みが必要 (`{ja: "...", en: "...", zh: "..."}`)。複雑になるので別案。

## 想定する対応タイミング

- 別プロジェクト (kawaz/* 以外) で利用要望が出たとき
- または 3+ 言語サポートが必要になったとき (v3.0.0 候補)

それまでは kawaz docs-structure.md 規約の hard-code で十分。

## 関連

- v2.1.0 で 3 task に分割 (commit-lag / links / umbrella)、CHANGELOG.md 参照
- kawaz docs-structure.md (相互リンクの規約定義)
