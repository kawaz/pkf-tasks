# pkf-tasks

> English | [日本語](./README-ja.md)

Common Pkl task modules for [mizchi/pkfire](https://github.com/mizchi/pkfire) (a typed task runner configured in Pkl). Consolidates the shared pkfire templates used across kawaz/\* repositories into one place.

## Design

- **abstract module + value-level selection**: an abstract module (`tasks/vcs/iface.pkl`) declares the interface; concrete implementations (`tasks/vcs/jj.pkl`, `tasks/vcs/git.pkl`) `amends` it; an entry module (`tasks/vcs/auto.pkl`) probes the filesystem and selects the right one at evaluation time.
- **Same interface across implementations**: jj and git implementations expose the same properties (`commit` / `push` / `fetch` / `ensureClean`) so callers reach for `vcs.push` transparently.
- **Distributed as a Pkl package**: `package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@<version>`. The Pkl cache (`~/.pkl/cache/package-2/`) handles SHA256 verification and offline replay.

## Usage

From any project's `Taskfile.pkl`:

```pkl
amends "package://pkg.pkl-lang.org/github.com/mizchi/pkfire/pkfire@0.4.0#/Taskfile.pkl"

import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.1#/vcs/auto.pkl" as vcs
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@0.0.1#/docs/translations.pkl" as docs

local push: Task = new {
  name = "push"
  deps { vcs.ensureClean; docs.checkTranslations }
  cmd = "..."
}

tasks { push; docs.checkTranslations; vcs.push }
```

## Modules

| Path | Contents |
|---|---|
| `vcs/iface.pkl` | abstract module (commit / push / fetch / ensureClean) |
| `vcs/jj.pkl`    | jj implementation (amends iface) |
| `vcs/git.pkl`   | git implementation (amends iface) |
| `vcs/auto.pkl`  | entry point (selects jj or git by filesystem probe) |
| `docs/translations.pkl` | Translation-pair integrity check (README / docs/DESIGN / docs/MANUAL, etc.) |

Planned: `go/` (gofmt / vet / test / build), `rust/` (cargo fmt / clippy / build / test), `release/` (bump-version family + push).

## Development

```bash
cd tasks
pkl eval vcs/auto.pkl       # smoke-test module loading
pkl project package         # build the package zip + metadata
```

Releases are published as GitHub Release zip uploads (`pkg.pkl-lang.org` proxies them).

## Related

- Conventions for consumers: `~/.claude/rules/docs-structure.md`
- pkfire itself: <https://github.com/mizchi/pkfire>
- Example consumer: <https://github.com/kawaz/bump-semver>
