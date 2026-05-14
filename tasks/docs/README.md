# `docs/` — document integrity checks

> English | [日本語](./README-ja.md)

Translation-pair integrity checks (`*-ja.md` / `*.md`) per the kawaz `docs-structure.md` convention — existence, mutual links, and commit-timestamp ordering. The expected workflow is to wire it into `push`'s deps so that "forgot the English translation" and "updated the English version but left the Japanese original behind" are caught at push time.

v2.1.0+ splits the work into three tasks (one umbrella + two sub-checks) and supports CLI args / neighbor discovery for multilingual use.

## Public entry vs internal files

- **Public entry**: `tasks/docs.pkl` — the only file consumers should `import`. Exported field names are part of the 2.x public API contract
- **Internal implementation** (do not import directly; renames/moves may happen in any minor release):
  - `translations.pkl` — the three Task definitions plus the `forPairs(pairs)` factory function. Re-exported from `tasks/docs.pkl`

## Provided tasks

### `docs:check-translations` (umbrella)

- Usefulness: ★★★
- Composition: `deps { docs:check-translation-commit-lag; docs:check-translation-links }` — runs both sub-checks in parallel
- Caveat: pkfire 0.6.0 orchestrator does **not** forward CLI args from umbrella to deps. If you need to pass `--` args, call the sub-checks directly

### `docs:check-translation-commit-lag`

- Usefulness: ★★★
- What it does: for each source file, compares the VCS commit timestamp against all of its neighbors (= translation targets). Fails if any translation is older than its source (= the translation hasn't been updated to match the source's latest edit).
- Multilingual-safe: a single source can have N translation targets; each is compared independently.

### `docs:check-translation-links`

- Usefulness: ★★★
- What it does: verifies the bilingual cross-link convention. Within the first 5 lines:
  - `<base>-ja.md` must contain `> [English](./<base>.md) | 日本語`
  - `<base>.md` must contain `> English | [日本語](./<base>-ja.md)`
- Bilingual only (1 source ↔ 1 target). Sources with 0 or 2+ neighbors log a skip and pass — the link convention hasn't been generalised for those cases (see `docs/issue/2026-05-12-link-pattern-injection.md` for the planned generalisation).

## Source selection

Three input modes, listed in priority order:

### 1. CLI args (`acceptsArgs = true`)

```bash
# explicit list
pkf run docs:check-translation-commit-lag -- README-ja.md docs/DESIGN-ja.md docs/MANUAL-ja.md

# glob (** requires bash 4+; bash 3.2 falls back to single-level *)
pkf run docs:check-translation-commit-lag -- '**/*-ja.md'

# bash 3.2 workaround for recursive glob
pkf run docs:check-translation-commit-lag -- {,*/,*/*/}*-ja.md
```

### 2. Pkl Listing (via `forPairs(...)`)

```pkl
import "package://pkg.pkl-lang.org/github.com/kawaz/pkf-tasks/pkf-tasks@3.0.0#/docs.pkl" as docs

local myCheck: Task = docs.forPairs(new Listing<String> {
  "README-ja.md"
  "docs/CONTRIBUTING-ja.md"
})

tasks { myCheck }
```

`task(pairs)` is kept as a `@Deprecated` alias for v2.x consumers; it will be removed in v3.0.

### 3. Auto-discover (default)

If no CLI args and the `pairs` Listing is empty, every `*-ja.md` under cwd is treated as a source. The following directories are excluded:

- `.jj/`
- `.git/`
- `*/.out/`
- `*/node_modules/`

## Neighbor discovery (source → translation targets)

For each source file, neighboring files with matching basename in the same directory are discovered as translation targets. The source itself is excluded.

| source pattern | targets discovered |
|---|---|
| `<base>-ja.md` (kawaz convention) | `<base>.md` only (1:1, en) |
| `<base>.md` (e.g. English-original) | `<base>-??.md` / `<base>-???.md` (2-3 letter language codes) |

- The `-ja.md` source uses **1:1 discovery** (en only) to avoid false positives like `data-layout-ja.md` matching `data-layout-history-ja.md`
- The `*.md` source restricts to short language-code suffixes (`-ja.md`, `-zh.md`, `-jpn.md`, etc.) for the same reason

## Design rationale

- **VCS commit timestamps, not stat mtime**: jj workspace switching makes stat mtime unstable. jj → `jj log -T 'committer.timestamp().format("%s")'`, git → `git log -1 --format=%ct -- <file>`
- **Untracked files fall back to ts=0**: `git log -1 --format=%ct -- <untracked>` exits 0 with empty stdout, so we normalise to `0` to avoid silent `[ "" -lt N ]` exit-2 errors in the lag comparison
- **Single Task per check (not `Listing<Task>` fan-out)**: when per-pair cache granularity isn't needed, one Task is enough — this is the pkfire convention
- **Hard-coded link pattern**: the link check assumes the kawaz `docs-structure.md` convention. Other projects can implement their own check task and `deps` it themselves; making the pattern injectable is tracked in `docs/issue/2026-05-12-link-pattern-injection.md`

## Related

- `docs/decisions/` — Decision Records
- `docs-structure.md` (kawaz personal rule) — the translation-pair convention this checks
