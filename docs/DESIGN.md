# pkf-tasks Design

> English | [日本語](./DESIGN-ja.md)

Internal architecture and design decisions for `kawaz/pkf-tasks`. End-user usage lives in [README.md](../README.md); per-release changes in [CHANGELOG.md](../CHANGELOG.md).

## Module layout

```
tasks/
├── PklProject               # Pkl package metadata (packageUri / dependencies / version)
├── PklProject.deps.json     # Resolved dependency lockfile (SHA256-pinned)
├── all.pkl                  # Root aggregator — single `import .../all.pkl as kawaz` entry point
├── vcs/
│   ├── iface.pkl            # abstract module (commit / push / fetch / ensureClean / fetchTags)
│   ├── jj.pkl               # jj implementation (extends iface)
│   ├── git.pkl              # git implementation (extends iface)
│   └── auto.pkl             # entry — Task export + Pkl function helpers (diffSummary / readAtRef)
├── docs/
│   └── translations.pkl     # README{,-ja}.md etc. pair check (single task, bash for loop)
├── lint/
│   ├── pkl.pkl              # language-agnostic `pkl format -w` task
│   ├── all-coverage.pkl     # `lint:all-coverage` — orphan .pkl module detector under tasks/
│   └── all.pkl              # lint/ sub aggregator
├── semver/
│   ├── check-bumped.pkl     # parameterized SemVer bump gate (bump-semver required)
│   ├── compare.pkl          # `semver:compare` — thin wrapper around bump-semver compare (acceptsArgs)
│   └── all.pkl              # semver/ sub aggregator
└── migrate/
    ├── check-current.pkl    # `migrate:check-pkf-tasks-current` — pkf-tasks import freshness gate
    ├── update-self.pkl      # `migrate:update-pkf-tasks` — rewrites the import in Taskfile.pkl
    └── all.pkl              # migrate/ sub aggregator
```

The internal module FQN uses `com.github.kawaz.pkfTasks.*`. Consumers always `import ... as <alias>`, so this rename is non-breaking; the reverse-domain notation just had to be corrected because `kawaz.com` is not owned (fixed in v0.0.6).

## Aggregate import — the two-tier `all.pkl` design

A single `import "package://...pkf-tasks@<version>#/all.pkl" as kawaz` line gives consumers access to every group (introduced in v0.0.10). Since every kawaz/* repo is expected to use the full pkf-tasks surface, an aggregate import is more DRY than wiring up individual `vcs/auto.pkl as vcs`, `docs/translations.pkl as docs`, ... imports.

```pkl
import "package://...pkf-tasks@0.0.12#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks                                       // commit / push / fetch / ensureClean / fetchTags
  kawaz.docs.checkTranslations
  ...kawaz.lint.allTasks                                      // pkl format + all-coverage
  kawaz.semver.compare                                        // ad-hoc CLI use (acceptsArgs)
  (kawaz.semver.checkBumped) { compareRefCmd = "..." }.check  // parameterize via module reference
  ...kawaz.migrate.allTasks                                   // check + update
}
```

The structure is two-tiered: the **root `tasks/all.pkl`** imports each group's `tasks/<group>/all.pkl` and re-exports it under a namespace, while each sub `all.pkl` bundles that group's modules.

### Task-direct vs. module-reference exports in each sub `all.pkl`

Each sub `all.pkl` mixes two export styles depending on intent:

- **Task-direct export** — `kawaz.lint.pkl` (a `Task`) can be registered as-is with `tasks { kawaz.lint.pkl }`. The ad-hoc CLI task `kawaz.semver.compare` (an `acceptsArgs` Task) follows the same shape.
- **Module-reference export** — `kawaz.semver.checkBumped` is exported as the **module** so consumers can parameterize it: `(kawaz.semver.checkBumped) { compareRefCmd = ...; versionFiles = ... }.check` produces an instance task.

Each sub `all.pkl` also exposes `allTasks: Listing<Task>` so consumers can use spread registration like `tasks { ...kawaz.lint.allTasks }` (added in v0.0.11). Parameterize-required modules are intentionally excluded from `allTasks` — instantiating them is the consumer's responsibility.

### `lint:all-coverage` — automating aggregator integrity via orphan detection

Hand-maintaining `all.pkl` is fragile (it's easy to add a new module and forget to wire it in). `lint:all-coverage` is the gate task that catches this in CI / pre-push (introduced in v0.0.10, detection logic improved in v0.0.11).

The detection rule is: **fail if any `.pkl` module under `tasks/` is not referenced from anywhere else** (an "orphan" check). The v0.0.10 implementation looked only at `all.pkl`-family references, which gave false positives for iface-implementation files like `vcs/jj.pkl` / `vcs/git.pkl` (imported by `vcs/auto.pkl` but never written into `all.pkl`). v0.0.11 relaxed the scope to "any `.pkl` under tasks/" and resolved the issue.

Exclusions: `PklProject*` (metadata), `Taskfile.pkl` (dogfood entry), `iface.pkl` (abstract; `extends` references are not visible to grep, so they're explicitly excluded), and `all.pkl` itself (aggregator endpoint referenced externally via package URLs).

Design choice: detection only for now. Auto-fix (regenerating `all.pkl`) is left for a future `fix:all-coverage` task. The detect/fix split mirrors `semver:check-version-bumped` (gate) vs. `bump-version` (action).

## VCS abstraction — abstract module + extends + runtime dispatch

`tasks/vcs/iface.pkl` declares the interface as an `abstract module`. `jj.pkl` / `git.pkl` provide implementations via `extends "iface.pkl"` — **not** `amends`, because Pkl abstract modules can only be implemented through `extends` (see [DR-0001](./decisions/DR-0001-abstract-module-extends.md)).

`auto.pkl` imports both jj and git modules and builds each task's `cmd` by stitching the two with a bash if-else via an `autoCmd(jjCmd, gitCmd)` helper. The VCS choice is **not** made at Pkl evaluation time; the generated cmd itself contains `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi`. This works around Pkl's pure-evaluation model, which cannot observe filesystem state from a relative path (see [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md)).

Detection order: `.jj` wins over `.git`. `jj root` and `git rev-parse --git-dir` are preferred over `[ -d .jj ]` / `[ -d .git ]` because the upstream CLIs walk upward to the repository root — so `jj workspace add`-style subdirectories without `.jj` directly under cwd still resolve correctly.

The Pkl object-amend syntax `(jj.commit) { description = ...; cmd = autoCmd(...) }` lets `auto.pkl` use jj's task as a base and override only `description` and `cmd`, inheriting `params` / `env` / `cache` / `shell` etc. unchanged from the jj side.

### Two-layer export — Task plus Pkl function helpers

`vcs/auto.pkl` exports **Tasks** (`commit` / `push` / `fetch` / `ensureClean` / `fetchTags`) **and** **Pkl functions** from the same module:

- `diffSummary(ref: String, paths: List<String>): String` — returns a bash command substitution (the body of `$(...)`) that emits the list of files changed in the given paths relative to `ref`.
- `readAtRef(ref: String, path: String): String` — returns a bash command substitution that emits a file's contents at the given `ref`.

These return **bash fragments**, not structured values. Consumers embed them via Pkl string interpolation in other tasks' `cmd` (e.g. `\(vcs.diffSummary("$ref", List("src/")))`), and the bash if-else inside dispatches between jj and git at runtime (auto-detect).

We deliberately kept these helpers inside `vcs/auto.pkl` rather than splitting them into a separate `vcs/helpers.pkl` module: that would duplicate the jj/git dispatch logic. Treating `auto.pkl` as a single gateway for everything auto-detect-related means consumers only need one `import ... as vcs` for both Tasks and helpers. See [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md).

### `vcs/*` as the knowledge-storage location for jj/git differences

When higher-level recipes (like `semver/check-bumped`) grew, the temptation was to inline "jj does X, git does Y" bash branches into each recipe. That leads to duplicated knowledge across recipes and bloated recipe authors who keep re-discovering the same jj/git pitfalls. v0.0.9 **explicitly positioned `vcs/{iface,jj,git,auto}.pkl` as the VCS dispatch layer + the jj/git knowledge storage** ([DR-0006](./decisions/DR-0006-vcs-as-knowledge-storage.md)): every operation whose jj/git commands diverge belongs here, so recipe authors never have to re-derive the dispatch.

Two absorption shapes:

- **Add as a Task** — when consumers want to attach it to their `deps` (ordering + cache-key participation).
- **Add as a Pkl function** — when consumers want to splice a bash `$(...)` fragment into their own `cmd` (string assembly).

The canonical example added in v0.0.9 is **`vcs:fetch-tags`** (driven by `abstract fetchTags: Task` on iface):

- jj: `jj git fetch || true; jj git import || true` — `jj git fetch` honours `remote.<name>.tagOpt`, so without that config it doesn't pull tags. `jj git import` is then needed to surface tags fetched into the underlying bare git into jj's view. Both are best-effort (`|| true`) so transient network or config issues don't block downstream tasks (e.g. a `semver:check-bumped` that can fall back to whatever local tags already exist).
- git: `git fetch --tags origin`.

Adding an `abstract` to iface forces every `extends "iface.pkl"` implementation (jj / git / auto) to provide it at evaluation time (the compile-time safety dividend of DR-0001). External consumers don't extend iface directly, so they are unaffected.

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

## SemVer group — `semver/check-bumped.pkl` + `semver/compare.pkl`

### `semver:check-bumped` — bump-omission gate

A gate Task that fails if `triggerPaths` have changed since a comparison ref but the VERSION files were not bumped. Typical uses: a pre-push guard (`compareRef = main@origin`) and a pre-release CI guard (`compareRef = the most recent v* tag`).

It is parameterized so the consumer can instantiate multiple tasks:

- `compareRefCmd: String` — body of a bash command substitution returning the comparison ref
- `triggerPaths: List<String>` — paths whose changes should trigger the check (default `src/`)
- `versionFiles: List<String>` — VERSION files that must be bumped (multiple allowed)
- `taskName: String` — task name (consumers usually instantiate two: `semver:check-version-bumped` and `semver:check-against-latest-release`)
- `needsFetchTags: Boolean` — when `true`, inserts `vcs.fetchTags` into the task's `deps` (added in v0.0.9). Enable this whenever the `compareRefCmd` resolves to a tag-derived ref (`git tag -l 'v*' ...`, `vcs:latest-tag()`, ...) so the local jj/git tag view is synced before comparison. Defaults to `false` for the common `main@origin` push-gate case.

The consumer uses Pkl's object-amend pattern, `(semverCheck) { ... }.check`, to fan out to several tasks. Internally the task uses `vcs.diffSummary` to detect changes in `triggerPaths`; if non-empty, it runs `bump-semver compare gt VERSION vcs:"$compare_ref":VERSION` for each VERSION file.

### `semver:compare` — ad-hoc CLI wrapper (v0.0.8)

A thin wrapper around `bump-semver compare` that forwards any `<OP> <INPUT> <INPUT>` via pkfire's `acceptsArgs = true`:

```
pkf run semver:compare -- gt VERSION 1.0.0
pkf run semver:compare -- lt Cargo.toml vcs:latest-tag():Cargo.toml
```

Because pkfire's `Task.deps: Listing<Task>` cannot pass arguments to dependencies, this task is **for ad-hoc CLI use only**. Composite gates like `semver:check-bumped` cannot reuse it through `deps` and continue to invoke `bump-semver compare` inline.

### bump-semver is mandatory — SemVer comparison fallback is intentionally omitted

If `bump-semver` is not on `PATH`, the task stops with `not implemented: install bump-semver`. We intentionally do **not** ship a pure-Pkl / bash fallback (e.g. `sort -V`):

- SemVer's prerelease (`-rc.1`) and build-metadata (`+sha.abc`) ordering is subtle; `sort -V` gives counter-intuitive results (e.g. `0.14.0-rc.1` vs `0.14.0`).
- `bump-semver` already implements semver.org-compliant ordering. Re-implementing it in bash would be duplicate work and a maintenance hazard.
- pkf-tasks' responsibility ends at VCS dispatch + task composition; SemVer comparison belongs to bump-semver.

Consumers should install it via `brew install kawaz/tap/bump-semver` (or equivalent), including in CI. `semver/*` depends on `vcs/*` in a one-way layering with no reverse dependencies. See [DR-0005](./decisions/DR-0005-vcs-helper-functions-and-semver-group.md).

## migrate group — keeping pkf-tasks itself up to date (v0.0.11+)

Prevents consumers from leaving their `pkf-tasks@<version>` import stale by combining a pre-push gate with a manual update action. The design philosophy is **"push deps = notification trigger"**, the same idiom as `semver:check-*`.

### Gate / action separation

- **`migrate:check-pkf-tasks-current`** (`migrate/check-current.pkl`) — gate task. Fails if the `pkf-tasks@<version>` import in the consumer's `Taskfile.pkl` is older than the latest release. Designed to be added to the `push` task's `deps`.
- **`migrate:update-pkf-tasks`** (`migrate/update-self.pkl`) — action task. The consumer runs this manually (`pkf run migrate:update-pkf-tasks`) when the gate fails. It rewrites the version in-place via `sed -i.bak` but **does not auto-commit** — diff review is the consumer's responsibility.

The gate/action split mirrors `semver:check-version-bumped` (gate) vs. `bump-version` (action). The intended loop: "stale pkf-tasks blocks push → consumer runs the action → reviews the diff → commits".

### Latest-tag lookup via bump-semver (consolidated in v0.0.12)

v0.0.11 shipped an interim implementation that fetched the remote's latest tag with a `git ls-remote --tags` + `awk | grep | sort -V | tail -1` bash pipeline. In v0.0.12 — after bump-semver v0.15.0 added support for the `vcs:latest-tag(<repo>)` ref schema — the implementation was migrated to route through bump-semver, restoring the **intended responsibility split** (bump-semver = VCS-aware SemVer comparison + ref schema; pkf-tasks = task composition):

- Latest-tag lookup: `bump-semver get vcs:latest-tag(kawaz/pkf-tasks)`.
- Comparison: `bump-semver compare ge <current> <latest>` — now a **SemVer comparison**, not a string match, so the gate passes when `current >= latest` (pre-release / build-metadata ordering follows semver.org).
- Pinning to an unreleased / RC version is now handled correctly.

The complex bash pipeline disappears, and knowledge ends up in the right places (VCS-aware SemVer ref schema → bump-semver; task composition → pkf-tasks). The shift is consistent with DR-0006's "vcs as knowledge storage" direction.

## Distribution — Pkl package + GitHub Release

`tasks/PklProject` declares package metadata:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "0.0.12"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

Tag naming follows mizchi/pkfire's pattern `pkf-tasks@<version>` (see [DR-0003](./decisions/DR-0003-tag-naming.md)). `pkl project package` produces zip + metadata json + SHA256 files; these are uploaded as assets to the matching GitHub Release. The `pkg.pkl-lang.org` proxy serves them to downstream `pkl eval` / `pkf list` invocations.

On the consumer side, the Pkl cache lives at `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/`, with SHA256 verification. Once fetched, the package is replayable offline even if the Pkl proxy is unavailable.

## CI / release automation

`.github/workflows/ci.yml`: on push to main or PRs, set up pkl + pkf via mizchi/pkfire's composite action (`mizchi/pkfire@<commit SHA>`), run `pkl eval` smoke tests on each module, confirm `pkl project package` succeeds, and self-dogfood the translation-pair check on this repo's own READMEs.

`uses:` references the action by **commit SHA**, not by tag (`mizchi/pkfire@pkfire@<version>`). The pkfire release tags contain `@` (e.g. `pkfire@0.6.0`), which breaks the GitHub Actions workflow parser at the workflow-file level — the run fails so early that no logs are even produced. SHA pinning sidesteps this entirely and is also a recommended supply-chain hardening practice. See [DR-0004](./decisions/DR-0004-pkfire-action-sha-pin.md).

`.github/workflows/release.yml`: triggered by `pkf-tasks@*` tag pushes. It runs `pkl project package` and uploads the four expected assets to the matching GitHub Release. The workflow fails fast if the tag's version doesn't match the version declared in `PklProject`, preventing accidental mismatches.

## Related

- Decision records index: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- Release history: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* docs structure convention: `~/.claude/rules/docs-structure.md`
