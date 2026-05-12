# `docs/` — document integrity checks

> English | [日本語](./README-ja.md)

A single Task that validates translation pairs (`*-ja.md` / `*.md`) per the kawaz `docs-structure.md` rule — existence, mutual links, and commit-timestamp ordering — in one shot. The expected workflow is to wire it into `push`'s deps so that "forgot the English translation" and "updated the English version but left the Japanese original behind" are caught at push time.

Implemented as a single Task using a bash for-loop (no `Listing<Task>` fan-out). When per-pair cache granularity isn't needed, one Task is enough — this is the pkfire convention. The internal bash function for fetching commit timestamps auto-dispatches between jj and git (same runtime-detection strategy as `tasks/vcs/auto.pkl`).

## Module layout

- `translations.pkl` — exports the translation-pair integrity-check Task plus a `task(pairs)` function for consumers who want to verify a different set of pairs

## Provided tasks

### `docs:check-translations`

- Usefulness: ★★★
- Internals: for each pair `<name>`, in order
  1. skip if `<name>-ja.md` doesn't exist (optional-pair support)
  2. verify `<name>.md` (English version) exists in the same directory
  3. require the fixed link `> [English](./<basename>.md) | 日本語` within the first 5 lines of `<name>-ja.md` (exact match via `grep -qF`)
  4. require the fixed link `> English | [日本語](./<basename>-ja.md)` within the first 5 lines of `<name>.md`
  5. commit-timestamp ordering: `ja_ts <= en_ts` (ensures the English translation wasn't pushed late)
- Default target pairs: `README` / `docs/DESIGN` / `docs/MANUAL` (kawaz docs-structure.md convention)
- Example: wire into `push`'s deps

```pkl
tasks {
  ...kawaz.docs.allTasks   // registers docs:check-translations
}
```

To override the pair set, build a separate Task with the `task(pairs)` function in `translations.pkl`:

```pkl
import "package://...pkf-tasks@0.0.16#/docs/translations.pkl" as translations
local myCheck = translations.task(new Listing<String> {
  "README"
  "docs/CONTRIBUTING"
})
tasks { myCheck }
```

- Bug history: in 0.0.4 the `for` loop's exit code didn't propagate individual pair failures and the Task always reported success. Fixed with `set -euo pipefail` plus a `failed` accumulator — each pair runs as `|| { ...; failed=1; }` inside the loop and we `exit $failed` at the end
- Design rationale: commit timestamps come from VCS logs rather than stat mtime because jj workspace switching makes stat mtime unstable. jj → `jj log -T 'committer.timestamp().format("%s")'`, git → `git log -1 --format=%ct -- <file>`

## Related

- [Top README](../../README.md)
- `~/.claude/rules/docs-structure.md` — origin of the translation-pair naming convention and link templates
- Related module: internally follows the jj/git runtime-dispatch pattern from `tasks/vcs/auto.pkl`
