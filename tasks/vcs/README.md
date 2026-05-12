# `vcs/` — jj/git auto-dispatched VCS primitives

> English | [日本語](./README-ja.md)

VCS tasks that work transparently on either jj- or git-managed repositories, plus a home for VCS knowledge that can't be delegated to other tools like `bump-semver` (DR-0006). The interface is extracted with the abstract-module + extends pattern (DR-0001), and `auto.pkl` dispatches at **runtime** (inside each Task's cmd) via `jj root` / `git rev-parse --git-dir` (DR-0002).

Pkl evaluates purely and cannot observe filesystem state, so picking jj.pkl or git.pkl at eval time is not viable. Instead we pull the cmd fragments from both jj.pkl and git.pkl and stitch them with a bash if-else. `.jj` takes precedence over `.git` (the kawaz git-bare + jj-workspace layout has both, but jj is authoritative).

## Public entry vs internal files

- **Public entry**: `tasks/vcs.pkl` — the only file consumers should `import`. Exported field names are part of the 1.x public API contract
- **Internal implementation** (do not import directly from outside; renames/moves may happen in any minor release):
  - `iface.pkl` — abstract module. Interface definitions for `commit` / `push` / `fetch` / `ensureClean` / `fetchTags` plus the `allTasks: Listing<Task>` aggregation property
  - `jj.pkl` — jj implementation (`extends "iface.pkl"`)
  - `git.pkl` — git implementation (`extends "iface.pkl"`)
  - `auto.pkl` — runtime-dispatch implementation. Uses each jj.pkl Task structure (params / env / cache) as a base and overrides cmd and description with the auto-detect variant. Also exports the `diffSummary` / `readAtRef` Pkl helper functions. Re-exported from `tasks/vcs.pkl`

## Provided tasks

### `vcs:commit`

- Usefulness: ★★★
- Args: `message` (required, commit message body)
- Internals: jj → `jj describe -m "$MESSAGE"` → `jj new`, git → `git commit -m "$MESSAGE"`
- Example: `pkf run vcs:commit -- --message='Release v0.1.0'`

### `vcs:push`

- Usefulness: ★★★
- Internals: jj → `jj bookmark set main -r @-` → `jj git push --bookmark main`, git → `git push origin main`
- Example: extend the consumer's `push` umbrella task with `(vcs.push) { deps { ... } }` to wire pre-push gates (lint / test / check-bumped, etc.) in front
- Caveat: leave the SSH agent socket to host inheritance (pkfire's `inheritEnv = true` default). Setting it explicitly via `env` makes `read("env:...")` resource-not-found in CI and fails eval

### `vcs:fetch`

- Usefulness: ★★
- Internals: jj → `jj git fetch`, git → `git fetch`

### `vcs:ensure-clean`

- Usefulness: ★★★
- Internals: jj → `[ "$(jj log -r @ --no-graph -T 'empty')" = "true" ]`, git → `[ -z "$(git status --porcelain)" ]`
- Example: wire into the `push` gates

### `vcs:fetch-tags`

- Usefulness: ★★
- Internals: jj → `jj git fetch || true` → `jj git import || true`, git → `git fetch --tags origin`
- Design rationale (DR-0006 knowledge storage): without `remote.<name>.tagOpt`, `jj git fetch` won't pick up tags. Combining with `jj git import` propagates tags fetched on the bare-git side back into jj. Both fall back with `|| true` for best-effort behaviour (network outages must not block downstream)
- Example: auto-deps for `semver:check-bumped` when `needsFetchTags = true`, and a prerequisite for `vcs:latest-tag()` style refs

## Provided Pkl helper functions (internal use)

Helpers that return a bash command-substitution snippet to be interpolated as `$(...)` inside another Task's cmd string.

### `vcs.diffSummary(ref: String, paths: List<String>): String`

Bash command substitution that returns the list of file names changed between an arbitrary ref and `@`. jj uses `jj diff -r "<ref>..@" --summary -- <paths>`, git uses `git diff --name-only "<ref>" -- <paths>`.

Example (inside a cmd):

```pkl
cmd = """
  diff_out=\(vcs.diffSummary("main@origin", List("src/")))
  [ -z "$diff_out" ] && exit 0
  # ... bump check etc.
"""
```

### `vcs.readAtRef(ref: String, path: String): String`

Bash command substitution that returns a file's contents at an arbitrary ref. jj → `jj file show -r <ref> -- <path>`, git → `git show <ref>:<path>`.

## Usage (whole group)

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.6.0#/Taskfile.pkl"
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/all.pkl" as kawaz

tasks {
  ...kawaz.vcs.allTasks   // bulk-register commit/push/fetch/ensure-clean/fetch-tags
}
```

Per-module import (vcs group only):

```pkl
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@1.0.0#/vcs.pkl" as vcs
tasks { vcs.push; vcs.ensureClean }
```

## Related

- [Top README](../../README.md)
- DR-0001 abstract-module + extends pattern
- DR-0002 runtime VCS dispatch
- DR-0005 placement of helper functions (`diffSummary` / `readAtRef`) in the vcs group
- DR-0006 vcs as knowledge storage
- DR-0007 group structure convention (flat `<group>.pkl` entry; internal `<group>/...pkl` files are not public API)
