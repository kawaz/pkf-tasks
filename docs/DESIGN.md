# pkf-tasks Design

> English | [日本語](./DESIGN-ja.md)

Internal architecture and design decisions for `kawaz/pkf-tasks`. End-user usage lives in [README.md](../README.md); per-release changes in [CHANGELOG.md](../CHANGELOG.md).

## Module layout

```
tasks/
├── PklProject               # Pkl package metadata (packageUri / dependencies / version)
├── PklProject.deps.json     # Resolved dependency lockfile (SHA256-pinned)
├── vcs/
│   ├── iface.pkl            # abstract module (commit / push / fetch / ensureClean)
│   ├── jj.pkl               # jj implementation (extends iface)
│   ├── git.pkl              # git implementation (extends iface)
│   └── auto.pkl             # entry — Task export + Pkl function helpers (diffSummary / readAtRef)
├── docs/
│   └── translations.pkl     # README{,-ja}.md etc. pair check (single task, bash for loop)
├── lint/
│   └── pkl.pkl              # language-agnostic `pkl format -w` task
└── semver/
    └── check-bumped.pkl     # parameterized SemVer bump gate (bump-semver required)
```

The internal module FQN uses `com.github.kawaz.pkfTasks.*`. Consumers always `import ... as <alias>`, so this rename is non-breaking; the reverse-domain notation just had to be corrected because `kawaz.com` is not owned (fixed in v0.0.6).

## VCS abstraction — abstract module + extends + runtime dispatch

`tasks/vcs/iface.pkl` declares the interface as an `abstract module`. `jj.pkl` / `git.pkl` provide implementations via `extends "iface.pkl"` — **not** `amends`, because Pkl abstract modules can only be implemented through `extends` (see [DR-0001](./decisions/DR-0001-abstract-module-extends.md)).

`auto.pkl` imports both jj and git modules and builds each task's `cmd` by stitching the two with a bash if-else via an `autoCmd(jjCmd, gitCmd)` helper. The VCS choice is **not** made at Pkl evaluation time; the generated cmd itself contains `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi`. This works around Pkl's pure-evaluation model, which cannot observe filesystem state from a relative path (see [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md)).

Detection order: `.jj` wins over `.git`. `jj root` and `git rev-parse --git-dir` are preferred over `[ -d .jj ]` / `[ -d .git ]` because the upstream CLIs walk upward to the repository root — so `jj workspace add`-style subdirectories without `.jj` directly under cwd still resolve correctly.

The Pkl object-amend syntax `(jj.commit) { description = ...; cmd = autoCmd(...) }` lets `auto.pkl` use jj's task as a base and override only `description` and `cmd`, inheriting `params` / `env` / `cache` / `shell` etc. unchanged from the jj side.

### Two-layer export — Task plus Pkl function helpers

`vcs/auto.pkl` exports **Tasks** (`commit` / `push` / `fetch` / `ensureClean`) **and** **Pkl functions** from the same module:

- `diffSummary(ref: String, paths: List<String>): String` — returns a bash command substitution (the body of `$(...)`) that emits the list of files changed in the given paths relative to `ref`.
- `readAtRef(ref: String, path: String): String` — returns a bash command substitution that emits a file's contents at the given `ref`.

These return **bash fragments**, not structured values. Consumers embed them via Pkl string interpolation in other tasks' `cmd` (e.g. `\(vcs.diffSummary("$ref", List("src/")))`), and the bash if-else inside dispatches between jj and git at runtime (auto-detect).

We deliberately kept these helpers inside `vcs/auto.pkl` rather than splitting them into a separate `vcs/helpers.pkl` module: that would duplicate the jj/git dispatch logic. Treating `auto.pkl` as a single gateway for everything auto-detect-related means consumers only need one `import ... as vcs` for both Tasks and helpers. See [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md).

## Translation-pair verification — single task, bash for loop

`tasks/docs/translations.pkl` checks pairs like `README{,-ja}.md` / `docs/DESIGN{,-ja}.md` / `docs/MANUAL{,-ja}.md` in one go. There is only one Pkl-level task definition; the bash `for pair in ...` loop inside processes each pair.

We do **not** explode this into a `Listing<Task>`. Translation checks don't need per-pair input-cache independence — invalidating one pair would just re-run the entire check, which is cheap. The pkfire idiom is "split into subtasks only when independent cache keys matter", and this case doesn't qualify.

The checks per pair:

1. Skip if `<pair>-ja.md` doesn't exist (so pairs are optional)
2. The English counterpart `<pair>.md` must exist in the same directory
3. `<pair>-ja.md` must contain `> [English](./<base>.md) | 日本語` in its first 5 lines
4. `<pair>.md` must contain `> English | [日本語](./<base>-ja.md)` in its first 5 lines
5. `ja_ts <= en_ts` for commit timestamps (the English translation must not lag behind)

The cmd uses `set -euo pipefail` plus an explicit `failed=0` accumulator and `exit $failed` at the end. This way **every pair is checked**, but **any single failure exits the task with status 1**. `set -e` alone would stop at the first failure and hide subsequent ones; the combination gives the best signal ([CHANGELOG 0.0.4](../CHANGELOG.md#004--2026-05-11)).

## Language-agnostic Lint — `lint/pkl.pkl`

`tasks/lint/pkl.pkl` exposes a `lint:pkl` task that runs `pkl format -w` recursively against `**/*.pkl`, `PklProject`, and `PklProject.deps.json`. Pkl formatting is a shared concern across kawaz/* repos, so it ships independently of language-specific lint (`gofmt` / `cargo fmt` / etc.).

Consumers wire it into an umbrella `lint` task alongside their language-specific lint tasks:

```pkl
local lint: Task = new {
  name = "lint"; cache = false
  deps { goLint; pklLint.format }
  cmd = "echo lint ok"
}
```

The module pins `minPklVersion = "0.31.0"` (same as pkfire) to keep `pkl format` behaviour consistent across kawaz/* repos.

## SemVer gate — `semver/check-bumped.pkl`

A gate Task that fails if `triggerPaths` have changed since a comparison ref but the VERSION files were not bumped. Typical uses: a pre-push guard (`compareRef = main@origin`) and a pre-release CI guard (`compareRef = the most recent v* tag`).

It is parameterized so the consumer can instantiate multiple tasks:

- `compareRefCmd: String` — body of a bash command substitution returning the comparison ref
- `triggerPaths: List<String>` — paths whose changes should trigger the check (default `src/`)
- `versionFiles: List<String>` — VERSION files that must be bumped (multiple allowed)
- `taskName: String` — task name (consumers usually instantiate two: `semver:check-version-bumped` and `semver:check-against-latest-release`)

The consumer uses Pkl's object-amend pattern, `(semverCheck) { ... }.check`, to fan out to several tasks. Internally the task uses `vcs.diffSummary` to detect changes in `triggerPaths`; if non-empty, it runs `bump-semver compare gt VERSION vcs:"$compare_ref":VERSION` for each VERSION file.

### bump-semver is mandatory — SemVer comparison fallback is intentionally omitted

If `bump-semver` is not on `PATH`, the task stops with `not implemented: install bump-semver`. We intentionally do **not** ship a pure-Pkl / bash fallback (e.g. `sort -V`):

- SemVer's prerelease (`-rc.1`) and build-metadata (`+sha.abc`) ordering is subtle; `sort -V` gives counter-intuitive results (e.g. `0.14.0-rc.1` vs `0.14.0`).
- `bump-semver` already implements semver.org-compliant ordering. Re-implementing it in bash would be duplicate work and a maintenance hazard.
- pkf-tasks' responsibility ends at VCS dispatch; SemVer comparison belongs to bump-semver.

Consumers should install it via `brew install kawaz/tap/bump-semver` (or equivalent), including in CI. `semver/*` depends on `vcs/*` in a one-way layering with no reverse dependencies. See [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md).

## Distribution — Pkl package + GitHub Release

`tasks/PklProject` declares package metadata:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "0.0.7"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

Tag naming follows mizchi/pkfire's pattern `pkf-tasks@<version>` (see [DR-0003](./decisions/DR-0003-tag-naming.md)). `pkl project package` produces zip + metadata json + SHA256 files; these are uploaded as assets to the matching GitHub Release. The `pkg.pkl-lang.org` proxy serves them to downstream `pkl eval` / `pkf list` invocations.

On the consumer side, the Pkl cache lives at `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/`, with SHA256 verification. Once fetched, the package is replayable offline even if the Pkl proxy is unavailable.

## CI / release automation

`.github/workflows/ci.yml`: on push to main or PRs, set up pkl + pkf via mizchi/pkfire's composite action (`mizchi/pkfire@<commit SHA>`), run `pkl eval` smoke tests on each module, confirm `pkl project package` succeeds, and self-dogfood the translation-pair check on this repo's own READMEs.

`uses:` references the action by **commit SHA**, not by tag (`mizchi/pkfire@pkfire@0.4.0`). The pkfire release tags contain `@` (`pkfire@0.4.0`), which breaks the GitHub Actions workflow parser at the workflow-file level — the run fails so early that no logs are even produced. SHA pinning sidesteps this entirely and is also a recommended supply-chain hardening practice. See [DR-0004](./decisions/DR-0004-pkfire-action-sha-pin.md).

`.github/workflows/release.yml`: triggered by `pkf-tasks@*` tag pushes. It runs `pkl project package` and uploads the four expected assets to the matching GitHub Release. The workflow fails fast if the tag's version doesn't match the version declared in `PklProject`, preventing accidental mismatches.

## Related

- Decision records index: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- Release history: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* docs structure convention: `~/.claude/rules/docs-structure.md`
