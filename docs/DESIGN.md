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
│   └── auto.pkl             # entry — concatenates jj/git cmds with a bash if-else
└── docs/
    └── translations.pkl     # README{,-ja}.md etc. pair check (single task, bash for loop)
```

## VCS abstraction — abstract module + extends + runtime dispatch

`tasks/vcs/iface.pkl` declares the interface as an `abstract module`. `jj.pkl` / `git.pkl` provide implementations via `extends "iface.pkl"` — **not** `amends`, because Pkl abstract modules can only be implemented through `extends` (see [DR-0001](./decisions/DR-0001-abstract-module-extends.md)).

`auto.pkl` imports both jj and git modules and builds each task's `cmd` by stitching the two with a bash if-else via an `autoCmd(jjCmd, gitCmd)` helper. The VCS choice is **not** made at Pkl evaluation time; the generated cmd itself contains `if jj root >/dev/null 2>&1; then ...; elif git rev-parse --git-dir >/dev/null 2>&1; then ...; fi`. This works around Pkl's pure-evaluation model, which cannot observe filesystem state from a relative path (see [DR-0002](./decisions/DR-0002-runtime-vcs-dispatch.md)).

Detection order: `.jj` wins over `.git`. `jj root` and `git rev-parse --git-dir` are preferred over `[ -d .jj ]` / `[ -d .git ]` because the upstream CLIs walk upward to the repository root — so `jj workspace add`-style subdirectories without `.jj` directly under cwd still resolve correctly.

The Pkl object-amend syntax `(jj.commit) { description = ...; cmd = autoCmd(...) }` lets `auto.pkl` use jj's task as a base and override only `description` and `cmd`, inheriting `params` / `env` / `cache` / `shell` etc. unchanged from the jj side.

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

## Distribution — Pkl package + GitHub Release

`tasks/PklProject` declares package metadata:

```pkl
package {
  name = "pkf-tasks"
  baseUri = "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/\(name)"
  version = "0.0.5"
  packageZipUrl = "https://github.com/kawaz/pkf-tasks/releases/download/\(name)@\(version)/\(name)@\(version).zip"
  ...
}
```

Tag naming follows mizchi/pkfire's pattern `pkf-tasks@<version>` (see [DR-0003](./decisions/DR-0003-tag-naming.md)). `pkl project package` produces zip + metadata json + SHA256 files; these are uploaded as assets to the matching GitHub Release. The `pkg.pkl-lang.org` proxy serves them to downstream `pkl eval` / `pkf list` invocations.

On the consumer side, the Pkl cache lives at `~/.pkl/cache/package-2/pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>/`, with SHA256 verification. Once fetched, the package is replayable offline even if the Pkl proxy is unavailable.

## CI / release automation

`.github/workflows/ci.yml`: on push to main or PRs, set up pkl + pkf via mizchi/pkfire's composite action, run `pkl eval` smoke tests on each module, confirm `pkl project package` succeeds, and self-dogfood the translation-pair check on this repo's own READMEs.

`.github/workflows/release.yml`: triggered by `pkf-tasks@*` tag pushes. It runs `pkl project package` and uploads the four expected assets to the matching GitHub Release. The workflow fails fast if the tag's version doesn't match the version declared in `PklProject`, preventing accidental mismatches.

## Related

- Decision records index: [docs/decisions/INDEX.md](./decisions/INDEX.md)
- Release history: [CHANGELOG.md](../CHANGELOG.md)
- kawaz/* docs structure convention: `~/.claude/rules/docs-structure.md`
