# `migrate/` — upstream-tracking pairs (gate + action)

> English | [日本語](./README-ja.md)

A group that detects whether the consumer's `Taskfile.pkl` is using `pkf-tasks@<version>` import / `pkfire@<version>` amends that are older than the latest upstream release, and rewrites them automatically when needed. **Two gate + action pairs per upstream target** = four tasks total. Requires the `bump-semver` CLI on PATH (uses `vcs:latest-tag(<repo>)` to fetch the latest tag).

The design philosophy is the same as `semver:check-*`: **push's deps = the discovery trigger**. Wire each `check-*` task into `push`'s deps so that a stale dependency pin fails at push time, and recover with the matching `update-*` task.

## Public entry vs internal files

- **Public entry**: `tasks/migrate.pkl` — the only file consumers should `import`. Exported field names (`checkPkfTasks` / `updatePkfTasks` / `checkPkfire` / `updatePkfire` / `allTasks`) are part of the 1.x public API contract
- **Internal implementation** (do not import directly from outside; renames/moves may happen in any minor release):
  - `check-current.pkl` — `migrate:check-pkf-tasks-current` (pkf-tasks tracking gate)
  - `update-self.pkl` — `migrate:update-pkf-tasks` (pkf-tasks auto-update action, sed in-place)
  - `check-pkfire-current.pkl` — `migrate:check-pkfire-current` (pkfire tracking gate)
  - `update-pkfire.pkl` — `migrate:update-pkfire` (pkfire auto-update action, wraps `pkf migrate`)

## Provided tasks

### `migrate:check-pkf-tasks-current` (pkf-tasks gate)

- Usefulness: ★★★
- Parameters (module attributes of `check-current.pkl`):
  - `taskfilePath: String` — default `"Taskfile.pkl"`
  - `remoteRepoSpec: String` — passed to bump-semver's `vcs:latest-tag(<arg>)`. default `"kawaz/pkf-tasks"`
  - `tagPrefix: String` — default `"pkf-tasks@"` (DR-0003)
- Internals:
  1. fetch the latest tag with `bump-semver get 'vcs:latest-tag(kawaz/pkf-tasks)'`
  2. extract `pkf-tasks@<x.y.z[-rc.1][+sha.abc]>` from `taskfilePath` with a regex covering full SemVer 2.0.0
  3. pass if `bump-semver compare ge <current> <latest>` is true, fail otherwise
- Example: place into `push`'s deps (`(vcs.push) { deps { ..., kawaz.migrate.checkPkfTasks } }`)

### `migrate:update-pkf-tasks` (pkf-tasks action)

- Usefulness: ★★
- Internals: fetch the latest tag with `bump-semver get`, then `sed -i.bak -E` (BSD/GNU compatible) rewrites `pkf-tasks@<version>` to the latest version in `taskfilePath`. No auto-commit (consumer reviews the diff and commits manually)
- Caveat: SemVer 2.0.0 regex (pre-release + build metadata). All matches in the file are replaced

### `migrate:check-pkfire-current` (pkfire gate)

- Usefulness: ★★★
- Parameters: `remoteRepoSpec` default `"mizchi/pkfire"` / `tagPrefix` default `"pkfire@"` (others same as `checkPkfTasks`)
- Internals: same structure as `migrate:check-pkf-tasks-current`. The only difference between the targets is `amends` (pkfire) vs `import` (pkf-tasks), and the tag-prefix grep operates with the same regex
- Separation of concerns: pkfire (`amends`) and pkf-tasks (`import`) are kept as separate tasks. Combining them would obscure which one is responsible when a failure happens

### `migrate:update-pkfire` (pkfire action)

- Usefulness: ★★
- Internals: fetch the latest tag with `bump-semver get`, then invoke `pkf migrate --to=<latest> --file=<taskfilePath>`
- Design rationale: instead of a homegrown sed implementation, use **`pkf migrate`** (built into pkfire 0.6.0+). The pkfire side rewrites the amends URI **with Pkl eval verification**, and **auto-reverts** if eval fails. Stricter than a homegrown sed
- Caveat: `pkf migrate` only targets **amends URIs**. It can't operate on `import` URIs (the pkf-tasks side is handled by the separate `update-pkf-tasks` task)

## Usage (whole group)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.migrate.allTasks   // register all 4 tasks
}

// wire the check-* tasks into push's deps
local push = (kawaz.vcs.push) {
  deps {
    kawaz.vcs.ensureClean
    kawaz.migrate.checkPkfTasks   // fail if pkf-tasks is behind
    kawaz.migrate.checkPkfire     // fail if pkfire is behind
  }
}
tasks { push }
```

Run them in bulk with glob targets (pkfire 0.6.0+):

```bash
pkf run 'migrate:check-*'    # run every check-* in one shot
pkf run 'migrate:update-*'   # recovery after a check failure (run every update-*)
```

## Naming convention and breaking changes

- **v0.0.16** renamed `kawaz.migrate.check` → `kawaz.migrate.checkPkfTasks` etc. to **symmetric naming**
- Up to v0.0.15 the pkf-tasks-facing names were implicit `check` / `update`, but adding `checkPkfire` / `updatePkfire` made the target ambiguous, hence the cleanup (DR-0007)
- Task names (`migrate:check-pkf-tasks-current` etc.) were already symmetric in v0.0.15. This change is **Pkl-side property names only**

## Related

- [Top README](../../README.md)
- DR-0003 tag naming (`pkf-tasks@<version>` / `pkfire@<version>`)
- DR-0007 group structure convention (flat `<group>.pkl` entry + symmetric naming; internal `<group>/...pkl` files are not public API)
- DR-0019 bump-semver's `vcs:latest-tag(<arg>)` ref schema (used for latest-tag retrieval in this group)
- Related CLIs: [`kawaz/bump-semver`](https://github.com/kawaz/bump-semver) (`brew install kawaz/tap/bump-semver`) / [`mizchi/pkfire`](https://github.com/mizchi/pkfire)'s `pkf migrate`
