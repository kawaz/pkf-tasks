# Changelog

All notable changes to `kawaz/pkf-tasks` are recorded here. The package follows [SemVer](https://semver.org/). Starting from **1.0.0** the module entry API is stable — breaking changes go in major bumps; minor/patch keep backward compatibility. Major-only pinning (`pkf-tasks@1` / `pkf-tasks@2`) is supported.

## 2.0.0 — 2026-05-12

A same-day major bump after a post-1.0.0 audit (the v1 release was less than 24 h old, the only consumer was `kawaz/bump-semver` and the `@1` pin had not yet been used in anger). The audit found that the `lint` group was library-internal to `pkf-tasks` itself rather than a generally reusable concern, so it leaked through what `1.0.0` had just declared as a stable public surface. Cutting `2.0.0` immediately while no real consumer depended on `@1` was cheaper than keeping the leak around for a future major.

### Breaking — `lint` group removed from the library

- **Removed** from library export: `tasks/lint.pkl`, `tasks/lint/pkl.pkl`, `tasks/lint/all-coverage.pkl`, and the entire `tasks/lint/` directory. The tasks themselves (`lint:pkl`, `lint:all-coverage`) were only ever useful to `pkf-tasks` itself; they were not consumed by any external project.
- The aggregator `tasks/all.pkl` no longer exposes `kawaz.lint.*`. Consumers that did `...kawaz.lint.allTasks` must drop that line.
- The dogfood equivalents have been **moved inline** into the repo-root `Taskfile.pkl` of `pkf-tasks` (not part of the library): `lint:pkl` (Pkl format) and a renamed `check:orphan-modules` (formerly `lint:all-coverage`, using the `check:` prefix to align with `semver:check-*` / `migrate:check-*` gate-task naming). The rename is purely internal to `pkf-tasks`'s own CI and does not affect consumers.

### Migration for consumers

Drop any reference to `kawaz.lint.*` / `pkf-tasks#/lint.pkl`:

```pkl
// Before (1.0.0):
import "package://.../pkf-tasks@1.0.0#/all.pkl" as kawaz
tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.lint.allTasks      // ← remove this line
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
}

// After (2.0.0):
import "package://.../pkf-tasks@2.0.0#/all.pkl" as kawaz
tasks {
  ...kawaz.vcs.allTasks
  ...kawaz.docs.allTasks
  ...kawaz.migrate.allTasks
  kawaz.semver.compare
}
```

If a project actually wants `pkl format -w` in its own pipeline, add it directly to its `Taskfile.pkl` (a 5-line `local lintPkl: Task = new { ... }`) instead of taking a library dependency for it.

### Stability statement (2.x)

Public surface that is **part of the 2.x contract** (breaking changes require a 3.x bump):

- `tasks/<group>.pkl` entry FQN + exported fields (task references + `allTasks` Listing + Pkl function helpers in vcs) — groups: `vcs` / `docs` / `semver` / `migrate`
- `tasks/all.pkl` bundled entry + `kawaz.<group>.<field>` namespace shape
- Public task names (`vcs:commit` / `docs:check-translations` / `semver:check-bumped` / `semver:compare` / `migrate:check-pkf-tasks-current` etc.)

Public surface that is **not part of the 2.x contract**: internal implementation files under each `<group>/` directory and CLI command bash internals (same as 1.x).

## 1.0.0 — 2026-05-12

The module entry API has stabilised after extensive 0.0.x iteration (17 releases). Going forward, **breaking changes go in major bumps**; the entry shape (`tasks/<group>.pkl` flat) and task names are part of the public contract.

### Breaking — flat group entry layout

- **Each group exposes a flat entry at `tasks/<group>.pkl`** (`vcs.pkl` / `docs.pkl` / `lint.pkl` / `semver.pkl` / `migrate.pkl`) instead of `<group>/all.pkl` (lint / semver / migrate) or `<group>/auto.pkl` (vcs) or `<group>/translations.pkl` (docs). The internal implementation files (`vcs/auto.pkl` etc.) are still present under each `<group>/` dir but are no longer considered the public entry — they are now implementation details.
- **Removed** (replaced by flat entries): `tasks/lint/all.pkl` / `tasks/semver/all.pkl` / `tasks/migrate/all.pkl`.
- **Consumer migration**:
  ```pkl
  // Before (0.0.x):
  import "package://.../pkf-tasks@<v>#/vcs/auto.pkl" as vcs
  import "package://.../pkf-tasks@<v>#/docs/translations.pkl" as docs
  import "package://.../pkf-tasks@<v>#/lint/all.pkl" as lint
  // (bundled all.pkl was already in place since 0.0.10)

  // After (1.0.0+):
  import "package://.../pkf-tasks@1.0.0#/vcs.pkl" as vcs
  import "package://.../pkf-tasks@1.0.0#/docs.pkl" as docs
  import "package://.../pkf-tasks@1.0.0#/lint.pkl" as lint
  // (bundled all.pkl works the same)
  ```
- `tasks/all.pkl` (bundled aggregation entry) is **unchanged** for consumers; the internal imports have moved from `<group>/all.pkl` to `<group>.pkl`, so `kawaz.vcs.*` / `kawaz.docs.*` / `kawaz.lint.*` / `kawaz.semver.*` / `kawaz.migrate.*` work identically.

### Stability statement

Public surface that is **part of the 1.x contract** (breaking changes require a 2.x bump):

- `tasks/<group>.pkl` entry FQN + exported fields (task references + `allTasks` Listing + Pkl function helpers in vcs)
- `tasks/all.pkl` bundled entry + `kawaz.<group>.<field>` namespace shape
- Public task names (`vcs:commit` / `docs:check-translations` / `lint:pkl` / `lint:all-coverage` / `semver:check-bumped` / `semver:compare` / `migrate:check-pkf-tasks-current` etc.)

Public surface that is **not part of the 1.x contract** (may change in minor/patch):

- Internal implementation files under `<group>/` directories (`vcs/iface.pkl` / `vcs/jj.pkl` / `vcs/git.pkl` / `vcs/auto.pkl` / `docs/translations.pkl` / `lint/pkl.pkl` / `lint/all-coverage.pkl` / `semver/check-bumped.pkl` / `semver/compare.pkl` / `migrate/*.pkl`) — consumers should not import these directly
- CLI command shapes inside `cmd` strings (bash internals)
- Default values for parameterised tasks (may evolve for better defaults)

## 0.0.17 — 2026-05-12

### Changed

- **`docs:check-translations` の default 検査対象を glob auto-discover に変更**。従来は明示列挙 (`README` / `docs/DESIGN` / `docs/MANUAL` の 3 ペア固定) だったが、`find . -name '*-ja.md'` で全ての `*-ja.md` を自動発見する形に。これにより `tasks/<group>/README-ja.md` のような新規ペアが追加されても **明示更新なしで自動的に検査対象になる**。明示指定したい場合は従来通り `task(new Listing { "README"; ... })` で別 Task を作れる (backward compatible)。
- 内部で `find` の除外パス: `.jj` / `.git` / `*/.out/*` / `*/node_modules/*`。

### Added

- **各 group ディレクトリに README ペア (`README.md` 英 + `README-ja.md` 日)** を追加。`tasks/vcs/` / `tasks/docs/` / `tasks/lint/` / `tasks/semver/` / `tasks/migrate/` の 5 group すべてに、各 task の詳細 (動作 / 引数 / 利用例 / 設計判断) を載せた group 内 README を配置。GitHub のディレクトリ表示で自動展開される動線が整い、リポトップ README から module 単位の詳細へ簡単に辿れる。
- リポトップ README を **module list 形式に簡潔化**。catalog テーブルを撤去し、5 module へのリンク + 短い 1 行説明に圧縮。詳細は各 module の README へ。

## 0.0.16 — 2026-05-12

### Breaking

- **`tasks/migrate/all.pkl` の export field を対称命名に rename**:
  - `kawaz.migrate.check` → `kawaz.migrate.checkPkfTasks`
  - `kawaz.migrate.update` → `kawaz.migrate.updatePkfTasks`
  - `checkPkfire` / `updatePkfire` はそのまま
  
  v0.0.15 で `checkPkfire` / `updatePkfire` を追加した時点で「`check` の対象が暗黙 (pkf-tasks)」が規約破綻していた。4 task 全てに対象 suffix を明示することで `pkf list` や利用側 Taskfile.pkl で誤読がなくなる。task name 自体 (`migrate:check-pkf-tasks-current` 等) は対象明示済なので変更なし、Pkl module の field 名のみ。
  
  **利用側の migration**:
  ```pkl
  // 旧 (0.0.15 まで)
  deps { kawaz.migrate.check }
  
  // 新 (0.0.16+)
  deps { kawaz.migrate.checkPkfTasks }
  ```

## 0.0.15 — 2026-05-12

### Added

- **`migrate:check-pkfire-current` (gate) + `migrate:update-pkfire` (action)** — pkfire 本体 (`amends` URI) の追従用 task ペア。pkf-tasks の追従用 (`migrate:check-pkf-tasks-current` / `migrate:update-pkf-tasks`) が `import` URI を対象にするのと役割分担。
  - check: `bump-semver get vcs:latest-tag(mizchi/pkfire)` で最新取得 + `bump-semver compare ge` で SemVer 比較 (`semver:check-bumped` と同じ流儀)
  - update: pkfire 0.6.0+ 内蔵の `pkf migrate --to=<latest>` をラップ。Pkl の post-migrate eval 検証つきで自前 sed よりも strict (eval fail なら自動 revert)
- **`migrate/all.pkl` を 4 task export に拡張** — `kawaz.migrate.{check,update,checkPkfire,updatePkfire}` + `allTasks` (4 task spread)。利用側で `push` の deps に `check` + `checkPkfire` を挟むと「pkf-tasks 古い / pkfire 古い」両方を一度に検知できる。

### Internal

- **release.yml が CHANGELOG.md から release notes を自動抽出** — 該当 `## <version>` section を `awk` で抽出して `gh release edit --notes-file` で上書き。手動の `gh release create --notes-file /tmp/...md` が不要に (pkfire の `scripts/release-notes.sh` 流儀を参考)。

## 0.0.14 — 2026-05-12

### Changed

- **pkfire dependency: `0.4.0` → `0.6.0`**。upstream pkfire の Pkl schema 自体は無変更だが、利用側 (kawaz/* リポ) の Taskfile.pkl も同じく `amends "package://...pkfire@0.6.0#/Taskfile.pkl"` に揃える必要 (Pkl の module version conflict 検出のため、root と zip 内で pkfire version が一致する必要)。

### Upstream notes

pkfire 0.6.0 で追加された CLI 機能 (`pkf affected --since=<ref>` / `pkf hooks install` / `pkf migrate --to=<ver>` / `pkf <plugin>` / `pkf completion` / glob target / `pkf cache *` etc.) は **Pkl schema 不変** のため pkf-tasks 側の Task 定義に影響なし。利用者は pkf binary を更新するだけで新機能を享受できる。

## 0.0.13 — 2026-05-12

### Fixed

- **`migrate:check-pkf-tasks-current` / `migrate:update-pkf-tasks` の version regex を SemVer 2.0.0 完全対応に拡張**。従来は `pkf-tasks@[0-9]+\.[0-9]+\.[0-9]+` のみマッチしていたため、将来 RC release (`pkf-tasks@0.1.0-rc.1`) を出した時に:
  - `check`: `current` が `0.1.0` として抽出され (実際は `0.1.0-rc.1`)、`bump-semver compare ge` が誤った判定を下す可能性
  - `update`: sed 置換が pre-release 部分を **完全消失** させる破壊的挙動 (`pkf-tasks@0.1.0-rc.1` → `pkf-tasks@<new>` 全置換)
  
  v0.0.13 で regex を `[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?(\+[0-9A-Za-z.-]+)?` に拡張、pre-release / build metadata を含めて検出 + 置換するように修正。セキュリティ点検 (2026-05-12) で指摘された将来検討事項への対応。

## 0.0.12 — 2026-05-12

### Changed

- **`migrate:check-pkf-tasks-current` / `migrate:update-pkf-tasks` の実装を bump-semver 経由に置換**。v0.0.11 は繋ぎ実装として `git ls-remote --tags` + `awk | grep | sort -V | tail -1` の bash pipeline で remote の最新 tag を取得していたが、bump-semver v0.15.0+ が `vcs:latest-tag(<repo>)` をサポートしたので、本来の責務分担 (bump-semver = VCS-aware SemVer 比較、pkf-tasks = task 合成) に整合させた。
  - `migrate:check-pkf-tasks-current` のロジック改善:
    - 最新取得: `bump-semver get vcs:latest-tag(kawaz/pkf-tasks)`
    - 比較: `bump-semver compare ge <current> <latest>` — 文字列マッチではなく **SemVer 比較** (current >= latest なら gate pass)
    - 利用側が未 release 版を pin している場合も適切に通る (semantic な比較)
  - `migrate:update-pkf-tasks` も `bump-semver get` 経由に統一、bash pipeline 削減
  - 新規パラメータ: `remoteRepoSpec: String = "kawaz/pkf-tasks"` (bump-semver の `vcs:latest-tag(<arg>)` に渡す repo spec、owner/repo 短縮形 or フル URL)
- **要件追加**: `bump-semver` v0.15.0 以上が PATH に必要 (`brew install kawaz/tap/bump-semver`)

## 0.0.11 — 2026-05-12

### Added

- **`tasks/migrate/check-current.pkl`** — `migrate:check-pkf-tasks-current` gate task。利用側 Taskfile.pkl の `pkf-tasks@<version>` import が最新 release より古いと fail させる。`push` task の deps に置いて「気づき発火点」として使う想定 (semver:check-* と同じ流儀)。`git ls-remote --tags` で remote の最新 tag を取得 (gh CLI 依存なし、git のみ)。
- **`tasks/migrate/update-self.pkl`** — `migrate:update-pkf-tasks` action task。`migrate:check-pkf-tasks-current` fail 時の復旧手段として利用者が手動で実行、`sed -i.bak` で Taskfile.pkl の import version を書き換える。自動 commit はせず diff 確認は利用者責任 (semver:check-version-bumped と bump-version の関係と同じ)。
- **`tasks/migrate/all.pkl`** — `migrate/` sub namespace 集約 (`kawaz.migrate.check` / `kawaz.migrate.update` / `kawaz.migrate.allTasks`)。
- **`tasks/lint/all.pkl` / `tasks/semver/all.pkl` に `allTasks: Listing<Task>`** — 利用側で `tasks { ...kawaz.lint.allTasks }` / `tasks { ...kawaz.semver.allTasks }` の spread 一括登録を可能化。

### Changed

- **`lint:all-coverage` の検出ロジック**: 「all.pkl 群限定の参照」から **「tasks/ 内のどこかで参照されていれば OK」(孤児検出)** に緩和。これにより `vcs/jj.pkl` `vcs/git.pkl` のような iface 実装ファイル (`vcs/auto.pkl` が `import` で参照、all.pkl には書かない) の false-positive 誤検出を解消。`tasks/all.pkl` 直 export か `<group>/all.pkl` 経由 export かは問わず、孤児にならなければ OK。

### Future

- bump-semver の `vcs:latest-tag(<repo>)` 機能 (`kawaz/bump-semver` docs/issue 起票済) が実装されたら、`migrate:check-pkf-tasks-current` / `migrate:update-pkf-tasks` の `git ls-remote` 実装を `bump-semver get vcs:latest-tag(kawaz/pkf-tasks)` 呼び出しに置換し、VCS-aware ref schema の責務を bump-semver に集約する予定。

## 0.0.10 — 2026-05-11

### Added

- **`tasks/all.pkl`** — root 集約エンドポイント。利用側で `import "package://...pkf-tasks@0.0.10#/all.pkl" as kawaz` 1 行で `kawaz.vcs.*` / `kawaz.docs.*` / `kawaz.lint.*` / `kawaz.semver.*` に透過アクセスできる。kawaz/* の各リポは全機能を使う前提なので、個別 import より集約 import の方が DRY。
- **`tasks/lint/all.pkl`** — `lint/` sub namespace 集約 (`lint:pkl` task と `lint:all-coverage` task を re-export)。
- **`tasks/semver/all.pkl`** — `semver/` sub namespace 集約 (`semver.checkBumped` を module 参照、`semver.compare` を task 直接で公開)。
- **`tasks/lint/all-coverage.pkl`** — `lint:all-coverage` task。`tasks/` 配下の `.pkl` module が `all.pkl` 群で全て re-export されているか検証し、漏れがあれば fail。これにより `all.pkl` の手動メンテ負債を CI / push 前 test で防止する (現状は検出のみ、`fix:all-coverage` の auto-fix は将来 task として追加予定)。

### Design

- 集約スタイルは「sub all.pkl は task を直接 export、parameterize 対象の module は module 参照で export」の方針。例: `kawaz.semver.checkBumped` は module 参照 (`(...).check` で parameterize)、`kawaz.semver.compare` は task 直接 (`tasks { kawaz.semver.compare }` で登録)。
- 利用者は依然個別 import (`import ".../vcs/auto.pkl" as vcs`) も使える。`all.pkl` は便宜の dependency endpoint であり obligation ではない。

## 0.0.9 — 2026-05-11

### Added

- **`vcs:fetch-tags` task** — abstracts the jj/git difference in tag fetching: jj does `jj git fetch || true; jj git import || true` (the latter syncs tags fetched into the bare git into jj's op log), git does `git fetch --tags origin`. Designed to be placed in the `deps` of any recipe that depends on the latest tag being visible (e.g. `vcs:latest-tag()` consumers).
- **`semver:check-bumped` parameter `needsFetchTags: Boolean = false`** — when `true`, inserts `vcs.fetchTags` into the gate task's `deps`. Use this for instances that compare against tag-derived refs (`git tag -l 'v*' ...`, `vcs:latest-tag()` etc.) so the local jj/git tag view is synced before the version comparison.

### Internal

- **`tasks/vcs/iface.pkl`**: added `abstract fetchTags: Task` (the type system forces jj/git/auto implementations to provide it — DR-0001's compile-time safety in action). External consumers do not `extends "iface.pkl"` directly so they are unaffected.
- **DR-0006**: codifies that `vcs/{iface,jj,git,auto}.pkl` is the **knowledge storage for jj/git differences** — every operation whose jj/git commands diverge belongs here, so recipe authors don't need to re-derive the dispatch each time. Sets a clear pattern for future additions.

## 0.0.8 — 2026-05-11

### Added (experimental)

- **`tasks/semver/compare.pkl`** — generic `semver:compare` task that wraps `bump-semver compare` and forwards `$@` via `acceptsArgs = true`. Designed for **ad-hoc CLI usage** (`pkf run semver:compare -- gt VERSION vcs:latest-tag():VERSION`). Cannot be composed via `deps` because pkfire's `Task.deps: Listing<Task>` does not support passing arguments to dependencies — so complex gates like `semver:check-bumped` continue to inline `bump-semver compare` rather than depending on `semver:compare`. See journal entry `2026-05-11-pkf-tasks-v0.0.8-semver-compare-experiment.md` for the experimental scope and trade-offs.

## 0.0.7 — 2026-05-11

> 0.0.6 is intentionally skipped — the changes accumulated on `main` after 0.0.5 (lint:pkl group, DR-0004 fix) are released together with the 0.0.7 namespace/helpers/bump additions in a single bundle. The release.yml workflow runs for the first time with this tag.

### Breaking (internal namespace, transparent to consumers)

- **Module FQN** changed from `com.kawaz.pkfTasks.*` to `com.github.kawaz.pkfTasks.*`. `kawaz.com` is not a domain owned by the author, so `com.github.kawaz` (Maven Central convention for hosted projects) is the correct reverse-domain notation. Consumers import via `import "..." as <alias>` and never reference the FQN directly, so caller-side code requires no changes. The rename touches all five vcs modules (`iface` / `jj` / `git` / `auto`), `docs/translations.pkl`, `lint/pkl.pkl`, and DR-0001.

### Added

- **`tasks/lint/pkl.pkl`** — language-agnostic Pkl `format -w` task (`lint:pkl`) covering `**/*.pkl`, `PklProject`, and `PklProject.deps.json`. Designed to plug into a consumer's `lint` umbrella task so language-specific (e.g. `lint:go`) and language-shared (`lint:pkl`) checks coexist under a single `lint` entry point.
- **`tasks/vcs/auto.pkl` helper functions** — `diffSummary(ref, paths)` and `readAtRef(ref, path)` now expose jj/git auto-dispatched bash command substitutions as Pkl function exports (returning bash `$(...)` fragments). Designed to be embedded inside other Task's `cmd` strings via `\(vcs.diffSummary(...))` interpolation. See DR-0005 for the rationale of mixing Task export + Pkl function export inside one module.
- **`tasks/semver/check-bumped.pkl`** — new `semver:` task group. Provides a parameterised `check` task with `compareRefCmd` / `triggerPaths` / `versionFiles` / `taskName` fields, so consumers can instantiate **two distinct check tasks from the same module**: one for push-time gating (compare against `main@origin`) and one for CI-time release gating (compare against latest `v*` tag). Requires `bump-semver` CLI (kawaz/tap/bump-semver) on PATH — when missing, the task exits with a "not implemented: install bump-semver" message rather than attempting a bash `sort -V` SemVer fallback (DR-0005). The group name `semver:` (domain-based) was chosen over `bump:` (action-based) to keep the door open for future siblings like `semver:bump-version` / `semver:compare`.

### Fixed

- **DR-0004**: real cause of the GHA composite action failure is `@` in the ref name (single-@ form `mizchi/pkfire@0.4.0` also fails), not the `@@` double. Decision (SHA pin) and migration steps remain the same, but the Context and switch-test table are now accurate.

## 0.0.5 — 2026-05-11

### Breaking

- **`tasks/vcs/jj.pkl` / `tasks/vcs/git.pkl`**: the `env { ["SSH_AUTH_SOCK"] = read("env:SSH_AUTH_SOCK") }` declaration on `push` / `fetch` tasks was **removed**. `read("env:...")` raises `Cannot find resource` during Pkl evaluation when the variable is unset, which made the modules un-evaluable in environments without an active SSH agent (CI runners, fresh shells). The `SSH_AUTH_SOCK` is now inherited from the host via pkfire's `inheritEnv = true` default, so callers no longer need to set it explicitly. Migration: nothing to do on the caller side; signing still works locally because the host shell exports `SSH_AUTH_SOCK`. CI environments that previously failed during `pkf list` / `pkl eval` will now succeed.

### Improved

- `tasks/vcs/auto.pkl`: `description` of each auto-selected task is now rewritten to "vcs commit/push/fetch/... (auto-detect: jj or git)" so callers do not see jj-specific wording on the dispatcher.
- `tasks/vcs/auto.pkl`: `indent` helper no longer emits trailing two-space lines for empty input rows (defensive improvement against future cmd edits).
- `README.md` / `README-ja.md`: corrected "amends" → "extends" mismatch (abstract modules in Pkl must be extended, not amended). Usage example version updated to `pkf-tasks@0.0.5`.

## 0.0.4 — 2026-05-11

### Fixed

- **`tasks/docs/translations.pkl`**: the bash `for` loop in the `docs:check-translations` task **did not propagate failures**. When `check_pair` failed inside the loop (e.g. translation timestamp out of order), bash continued to the next iteration and the loop's overall exit code was the last iteration's return value. Since `docs/MANUAL` typically has no `-ja.md` and `check_pair` returns 0 (skip), the gate **silently passed even when README or DESIGN pairs failed**. Fixed by adding `set -euo pipefail` and an explicit `failed=0` accumulator with `exit $failed` at the end. Affects every consumer that uses `docs.checkTranslations` (notably `kawaz/bump-semver`) — the gate is now actually enforced.

## 0.0.3 — 2026-05-11

### Fixed

- **`tasks/vcs/auto.pkl` / `tasks/docs/translations.pkl`**: VCS detection switched from `[ -d .jj ]` / `[ -d .git ]` to `jj root >/dev/null 2>&1` / `git rev-parse --git-dir >/dev/null 2>&1`. The old pattern only inspected the current directory and produced false negatives inside `jj workspace add`-style subdirectories where `.jj` lives at the workspace root (not under cwd). The new pattern uses the upstream CLIs' built-in upward search.

## 0.0.2 — 2026-05-11

### Breaking

- **`tasks/vcs/iface.pkl`**: `allTasks: Listing<Task>` was changed from `local` (module-private) to public so callers can write `tasks { ...vcs.allTasks }`. In 0.0.1 the property existed but was not externally accessible.

## 0.0.1 — 2026-05-11

### Added

- Initial release. Provides:
  - `tasks/vcs/{iface,jj,git,auto}.pkl`: abstract module + value-level selection between jj and git, dispatched at runtime inside cmd by probing the filesystem.
  - `tasks/docs/translations.pkl`: integrity check task for paired `*-ja.md` / `*.md` documents (existence, mutual link, commit-timestamp order).
- Distributed via GitHub Releases → `pkg.pkl-lang.org` proxy, with SHA256-verified zip + metadata.
