# `lint/` — language-agnostic lint + meta lint

> English | [日本語](./README-ja.md)

Provides lint that's common to any project (Pkl format) plus a self-consistency check for the pkf-tasks library itself (orphan-module detection). Language-specific lint (gofmt / cargo fmt / rustfmt, etc.) lives outside this group; consumers wire `lint:pkl` into the deps of their own umbrella lint task.

## Module layout

- `pkl.pkl` — `lint:pkl` task (runs `pkl format -w` over every Pkl file)
- `all-coverage.pkl` — `lint:all-coverage` task (library-developer use, orphan-module detection)
- `all.pkl` — group sub-namespace aggregation (accessible as `kawaz.lint.pkl` / `kawaz.lint.allCoverage`)

## Provided tasks

### `lint:pkl`

- Usefulness: ★★★
- Internals: recursive `pkl format -w .` from cwd
- inputs: `**/*.pkl` / `**/PklProject` / `**/PklProject.deps.json` (pkfire doublestar syntax)
- Example: wire into the consumer's `lint` umbrella task deps

```pkl
import "package://...pkf-tasks@0.0.16#/all.pkl" as kawaz
local lint = new Taskfile.Task {
  name = "lint"
  deps { goLint; kawaz.lint.pkl }
  cmd = "echo lint ok"
}
tasks { lint; goLint; kawaz.lint.pkl }
```

- Caveat: pkl 0.31+ required (same as pkfire's `minPklVersion`). To format only specific files, override with `(format) { cmd = "pkl format -w specific.pkl" }`

### `lint:all-coverage`

- Usefulness: ★ (library-developer use, normal consumers don't need it)
- Internals: any `.pkl` under `tasks/` whose basename is never referenced from another `.pkl` is flagged as "orphan" and fails the task. Implemented with `find ... -exec grep -l ...`; a single reference is enough to pass
- Excluded: `PklProject*` / `Taskfile.pkl` / `iface.pkl` / `all.pkl`
- Design rationale: at introduction (0.0.10) it required references "from all.pkl files only", but that produced false positives for iface implementations like `vcs/jj.pkl` (only `auto.pkl` imports them, never registered directly in all.pkl). 0.0.11 relaxed it to "any `.pkl` reference"
- Design rationale: detection only — the auto-fix counterpart (`fix:all-coverage`) is planned as a future separate task. Separating the gate from the action mirrors `semver:check-bumped` vs `bump-version` and `migrate:check-*` vs `migrate:update-*`

## Usage (whole group)

```pkl
tasks {
  ...kawaz.lint.allTasks   // registers both lint:pkl and lint:all-coverage
}
```

Consumer projects typically only need `kawaz.lint.pkl` inside their own umbrella deps. `lint:all-coverage` is used by pkf-tasks itself as dogfooding (CI gate).

## Related

- [Top README](../../README.md)
- DR-0007 group convention (`<group>/all.pkl` aggregation + sub-namespace exposure pattern)
