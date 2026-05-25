---
title: release.yml semver gate に latest-release 並列 check 追加 (DR-0039 canonical 同期、tag prefix 考慮)
status: open
category: request
created: 2026-06-28T20:07:08+09:00
last_read:
open_entered: 2026-06-28T20:07:08+09:00
wip_entered:
blocked_entered:
pending_entered:
discarded_entered:
resolved_entered:
discard_reason:
pending_reason:
close_reason:
blocked_by:
origin: bump-semver dogfood 報告 (session 911732b3、2026-06-28)
---

# release.yml semver gate に latest-release 並列 check 追加 (DR-0039 canonical 同期、tag prefix 考慮)

## 概要

bump-semver canonical (DR-0039) で更新された release.yml semver gate pattern を本リポに同期する。本リポは `pkf-tasks@<ver>` の tag prefix を持つ B 型変種で、canonical 同期時に tag prefix 考慮が必要。

## 背景

bump-semver canonical (DR-0039) で release.yml の semver gate pattern が更新された。本リポは `vcs:latest-tag()` + `gh release view "pkf-tasks@..."` の B 型変種で、tag 命名規約 (`pkf-tasks@<ver>`) が他リポと違うため canonical 同期時に注意要。

## 現状 (release.yml L60-89 該当)

`compare gt "$CURRENT" "$(bump-semver get vcs:latest-tag())"` + `gh release view "pkf-tasks@${CURRENT}"` のみ。`latest-release` 並列 check 無し。

## 修正方針

`latest-release` 並列 check を追加。pkf-tasks 固有の tag prefix を考慮:

- `vcs get latest-tag` の対象も `pkf-tasks@` prefix の tag に絞る必要あり (= `--include-prerelease` だけだと全 tag を見るので、別途 filter 検討)
- `gh release view` も同じ tag prefix で
- monorepo style tag は DR-0019 / DR-0032 の `@` peel fallback 経路で扱われる

canonical pattern は bump-semver の release.yml と DR-0039 参照、pkf-tasks 固有の tag prefix 解決は別途設計判断。

## 受け入れ条件

- [ ] `latest-release` 並列 check が release.yml に追加されている
- [ ] `pkf-tasks@` tag prefix が正しく考慮されている
- [ ] bump-semver DR-0039 の canonical pattern との整合が取れている

## 参考

- bump-semver の `.github/workflows/release.yml`
- bump-semver の docs/decisions/DR-0039-release-yml-semver-gate-pattern.md
- bump-semver の DR-0019 / DR-0032 (monorepo tag prefix 対応)
- kawaz/die dogfood 報告: session 911732b3、2026-06-28

## 優先度

中 (= B 型、tag prefix 考慮で実装に追加考慮要)。bump-semver v0.43.0 release 後に着手推奨。
