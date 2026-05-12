# 2026-05-12 — flat group entry の副次効果: import 行が pkl format の 100 文字制限内に収まる

## 観察

v1.0.0 で各 group の entry を `tasks/<group>.pkl` に flat 化したことで、利用側 Taskfile.pkl の `import` 行が `pkl format -w` の改行閾値 (~100 文字) 内に収まるようになった。

| import 形 | 行長 | format 挙動 |
|---|---|---|
| `import "package://...pkf-tasks@1.0.0#/all.pkl" as kawaz` | 96 文字 | 1 行で維持 |
| `import "package://...pkf-tasks@1.0.0#/docs.pkl" as docs` | 96 文字 | 1 行で維持 |
| 旧 `import "package://...pkf-tasks@0.0.X#/docs/translations.pkl" as docs` | 109 文字 | URI を別行に折り返し |

旧形式 (`<group>/<sub>.pkl`) は format で:

```pkl
import
  "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.X#/docs/translations.pkl"
  as docs
```

のように 3 行になっていた。新形式 (`<group>.pkl`) では 1 行に収まる。

## 設計判断との関係

v1.0.0 の flat 化 (`<group>/all.pkl` → `<group>.pkl`) は **「entry 命名の統一感」** を主目的にした (Option A、Option B の `<group>/all.pkl` 全統一案より flat を採用)。

副次効果として:
- path が 8〜14 文字短縮 (`docs/translations.pkl` 22 → `docs.pkl` 8、`vcs/auto.pkl` 12 → `vcs.pkl` 7、`lint/all.pkl` 12 → `lint.pkl` 8 等)
- 利用側の `import` 行が **100 文字制限内** に収まる
- `pkl format -w` で 1 行のまま維持され、視認性が高い

これは設計時には意識していなかった bonus。命名の **「短く意味が通る」** という美徳が、format ルールにも自然に乗る形になった。

## 教訓

ファイル名は **短く意味が通る** を目指すと、ツールチェーン (format / lint / diff 等) との相性も自然に良くなる。逆に深いネスト + 長い名前は path が伸びて format ルールに引っかかりやすい。

## 関連

- v1.0.0 release (flat entry 採用): CHANGELOG.md
- DR-0007 (group 構造規約): flat 化前の sub `all.pkl` 規約。v1.0.0 で `<group>.pkl` flat に進化
- pkl format の挙動: 観察ベース (公式 docs に明示の閾値記載は未確認、~100 文字が経験則)
