# `semver/` — SemVer gates + ad-hoc compare

> English | [日本語](./README-ja.md)

Provides SemVer-comparison gates and a CLI wrapper. The `bump-semver` CLI is required on PATH (DR-0005). pkf-tasks' responsibility ends at VCS dispatch + task composition; SemVer comparison is delegated to bump-semver. A fallback (re-implementing SemVer comparison in Pkl) is explicitly not adopted.

Two main use cases:

- **`semver:check-bumped`** — pre-push / pre-release gate against forgotten version bumps. Exposed as a **module reference** so consumers can parameterise it per-instance via `object-amends`
- **`semver:compare`** — thin wrapper around `bump-semver compare` (`acceptsArgs = true`), strictly for ad-hoc CLI use

## Public entry vs internal files

- **Public entry**: `tasks/semver.pkl` — the only file consumers should `import`. Exported field names (`checkBumped` module reference, `compare` task, `tasks`) are part of the 1.x public API contract
- **Internal implementation** (do not import directly from outside; renames/moves may happen in any minor release):
  - `check-bumped.pkl` — parameterisable gate module. Instantiate via `(kawaz.semver.checkBumped) { ... }.check`
  - `compare.pkl` — `semver:compare` task (ad-hoc CLI)

## Provided tasks

### `semver:check-bumped`

- Usefulness: ★★★
- Kind: **module reference** (consumer is expected to parameterise and instantiate; not included in `tasks`)
- Internals: detect whether `triggerPaths` changed relative to the compare ref using `vcs.diffSummary`; if changes are present, run `bump-semver compare gt <file> vcs:<ref>:<file>` for every entry in `versionFiles`. If any file is not bumped, fail
- Parameters:
  - `compareRef: String` — a plain VCS ref string (e.g. `main@origin`, `origin/main`, or a function call like `vcs:latest-tag(kawaz/pkf-tasks)`). default `"main@origin"` (pre-push guard, v3.0+). Earlier releases accepted a shell-command body via `compareRefCmd`; that was removed in v3.0 because consumers tended to put 10-line scripts in it instead of an actual ref
  - `triggerPaths: List<String>` — paths to watch for changes. default `List("src/")`
  - `versionFiles: List<String>` — files to bump (e.g. `VERSION` / `Cargo.toml` / `package.json`). default `List("VERSION")`
  - `taskName: String` — task name. Use distinct names when creating multiple instances. default `"semver:check-version-bumped"`
  - `needsFetchTags: Boolean` — when true, adds `vcs.fetchTags` as a dep. Enable when using tag-based refs (`git tag -l ...` / `vcs:latest-tag()`). default `false`
- Design rationale: when `bump-semver` is missing, the task aborts with `ERROR: bump-semver not installed. SemVer comparison fallback is not implemented.` (DR-0005). pkf-tasks deliberately doesn't carry its own SemVer comparison logic

### `semver:compare`

- Usefulness: ★★
- Kind: direct task (`acceptsArgs = true`, args after `--` pass through to bump-semver)
- Internals: runs `bump-semver compare "$@" --no-hint`
- Examples:
  - `pkf run semver:compare -- gt VERSION 1.0.0`
  - `pkf run semver:compare -- lt Cargo.toml vcs:latest-tag():Cargo.toml`
  - `pkf run semver:compare -- eq package.json vcs:main@origin:package.json`
- `<INPUT>` formats (inherited from bump-semver): `FILE` (basename auto-detected) / raw semver / `-` (stdin) / `vcs:REV[:FILE]` / `vcs:latest-tag()`
- Design rationale: pkfire's `deps` is `Listing<Task>` and can't pass arguments, so this task is strictly for ad-hoc CLI use. Composite gates like `check-bumped` cannot reuse it as a dep, so they invoke `bump-semver compare` directly in their own implementation
- Caveat: `cache = true` (pkfire folds `$@` args into the cache key). If a moving tag changes the result, that's the consumer's responsibility

## Usage (whole group)

Only `semver:compare` goes through `tasks`; `check-bumped` is instantiated:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.10.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/all.pkl" as kawaz

// pre-push guard (compare against main@origin)
local checkPush = (kawaz.semver.checkBumped) {
  compareRef = "main@origin"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")
  taskName = "semver:check-version-bumped"
}.check

// pre-release guard (compare against the latest semver tag via bump-semver's vcs: function)
local checkRelease = (kawaz.semver.checkBumped) {
  compareRef = "vcs:latest-tag()"
  triggerPaths = List("src/")
  versionFiles = List("VERSION")
  taskName = "semver:check-against-latest-release"
  needsFetchTags = true
}.check

tasks {
  ...kawaz.semver.tasks   // registers semver:compare only
  checkPush                  // attach the instance tasks individually
  checkRelease
}
```

`semver:check-bumped` is excluded from `tasks` because a meaningful instance can't be built without knowing the consumer project's version-file paths (the default is `VERSION`, but `Cargo.toml` etc. require parameterisation).

## Related

- [Top README](../../README.md)
- DR-0005 semver group, decision not to adopt a bump-semver fallback
- DR-0007 group structure convention (flat `<group>.pkl` entry; internal `<group>/...pkl` files are not public API)
- DR-0019 bump-semver's `vcs:latest-tag(<arg>)` ref schema (used for latest-tag retrieval in `migrate:check-*`)
- Related module: internally uses `diffSummary` from the vcs group
- Related CLI: [`kawaz/bump-semver`](https://github.com/kawaz/bump-semver) (`brew install kawaz/tap/bump-semver`)
