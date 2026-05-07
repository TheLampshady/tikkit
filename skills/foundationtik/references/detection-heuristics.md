# foundationtik — Detection Heuristics

Shell-only methods for computing each of the seven signals. No SonarQube, no custom collectors — just `git`, `grep`, `wc`, and shell pipelines.

Every heuristic has known blind spots. The skill should flag low-confidence detections in the resulting ticket's Evidence section so the reader can verify before acting. Each section below names the limits explicitly.

## Conventions

- `<path>` — the foundation's primary file or directory (e.g. `src/auth/index.ts`, `apps/api/src/cache.py`).
- `<window>` — a git range or `--since` window. Defaults: 30 commits or `--since='90 days ago'` unless the user overrides.
- `<consumer-glob>` — the project's source root for grepping consumers (e.g. `src/`, `apps/`).
- Shell examples assume bash/zsh on a POSIX system. Adjust quoting for fish/PowerShell as needed.

## 1. God class — `foundation-bloat`

**Lines of code:**

```bash
wc -l <path>
```

**Public method count (Python):**

```bash
# Count lines that look like top-level public method/function defs.
# 4-space indent = class body. No leading underscore = public.
grep -E '^    def [^_]' <path> | wc -l
# Plus module-level public functions:
grep -E '^def [^_]' <path> | wc -l
```

**Public exports (TypeScript / JavaScript):**

```bash
grep -cE '^export ' <path>
```

**Public methods (Go):**

```bash
# Capitalized identifier after func or func (recv) = exported
grep -cE '^func [A-Z]|^func \([^)]+\) [A-Z]' <path>
```

**Limits to flag:**
- Python: misses `@property`, `__call__`, dunder methods that are effectively public, and class-level public attributes. If the foundation is class-heavy, the LOC signal is more reliable than the method count.
- TypeScript: doesn't distinguish `export type` from `export function`. For a tighter count, filter further: `grep -cE '^export (function|class|const)'`.
- Go: misses public struct fields. Pair with a Ce check if you suspect leakage there.
- Multi-file foundations (a directory): run `wc -l <dir>/*` and sum, or use `tokei`/`scc` if available.

## 2. Untested surface — `foundation-untested-api`

**Step 1: list public methods/exports.** Use the per-language grep above to extract names, not just counts.

```bash
# Python public methods (returns names):
grep -oE '^    def [^_(][a-zA-Z_0-9]*' <path> | awk '{print $2}'

# TypeScript public exports (returns names):
grep -oE '^export (function|class|const) [a-zA-Z_0-9]+' <path> | awk '{print $NF}'
```

**Step 2: for each name, check whether any test references it.**

```bash
NAME="<method-name>"
grep -rl "$NAME" tests/ test/ __tests__/ 2>/dev/null
# If empty: untested.
```

**Limits to flag:**
- A method exercised indirectly via integration tests (test calls a higher-level API that internally calls this method) won't be detected. Mark these as low-confidence.
- Names like `get` or `init` collide across modules — for short names, scope the grep to imports of the foundation: `grep -rl "from <foundation-path> import .*$NAME" tests/`.
- TypeScript barrel exports can hide which file owns a name. If the foundation is barrel-exported from a parent path, run the test grep against the parent path too.

## 3. Wrong abstraction — `foundation-wrong-abstraction`

The signal is *growth* in parameter count and conditional density on the foundation's primary file across recent commits. We don't compute absolute thresholds; we compare an old commit to the current state.

**Parameter count growth:**

```bash
# Sum of arity across all defs at HEAD:
grep -oE 'def [a-zA-Z_0-9]+\([^)]*\)' <path> | awk -F',' '{print NF}' | paste -sd+ | bc

# Same at HEAD~30:
git show HEAD~30:<path> | grep -oE 'def [a-zA-Z_0-9]+\([^)]*\)' | awk -F',' '{print NF}' | paste -sd+ | bc
```

**Conditional growth:**

```bash
# Count if/elif/else/switch at HEAD:
grep -cE '^\s*(if |elif |else|switch |case )' <path>

# At HEAD~30:
git show HEAD~30:<path> | grep -cE '^\s*(if |elif |else|switch |case )'
```

**Net additions over a window (a leading indicator):**

```bash
git log --follow -p HEAD~30..HEAD -- <path> \
  | grep -cE '^\+.*(if |elif |def |function )'
```

**Limits to flag:**
- Counting params via splitting on `,` mishandles default values that contain commas (e.g. `x=(1, 2)`). For Python with type hints, use `ast`-based parsing if precise counts matter.
- A net-additions count over a window can't tell behaviour-preserving refactors from real growth. Treat the signal as "look here," not "this is wrong."
- Multi-line function signatures break naive single-line greps. If the foundation uses long signatures, run the count on a normalized form (e.g. via `ruff format --line-length=200`).

## 4. Shotgun surgery — `foundation-shotgun-surgery`

A single commit on the foundation's API that touched ≥ 5 consumer files in the same commit.

**List recent commits that touched the foundation:**

```bash
git log --since='90 days ago' --format='%H %s' -- <path>
```

**For a candidate commit, count the consumer files touched in the same commit:**

```bash
SHA="<commit-sha>"
# Files in the commit that are NOT the foundation itself:
git show --name-only --format='' "$SHA" | grep -v "^<path>$" | wc -l
```

**Cross-check that those files are actual consumers (import the foundation):**

```bash
git show --name-only --format='' "$SHA" \
  | grep -v "^<path>$" \
  | xargs -I{} sh -c 'grep -l "<foundation-import-pattern>" "{}" 2>/dev/null' \
  | wc -l
```

**Limits to flag:**
- A commit that legitimately changes both the foundation and unrelated files (rare but possible) inflates the count. Cross-checking against actual imports filters this out.
- Squash-merge workflows compress N small commits into one — this skews the heuristic toward "everything is shotgun surgery." If the project squashes, look at PR-level rather than commit-level changes (`gh pr view <num> --json files`).
- A foundation re-exported via a barrel will hide the import pattern. Use the barrel path instead, or both.

## 5. Coupling regression — `foundation-coupling`

**Afferent coupling Ca (incoming — how many consumers):**

```bash
# Files in the project that import the foundation:
grep -rl "from '<foundation-path>'" <consumer-glob> | wc -l
# Or for Python:
grep -rl "from <foundation-module> import" <consumer-glob> | wc -l
```

**Efferent coupling Ce (outgoing — how many things the foundation imports):**

```bash
# Distinct external imports in the foundation file (TypeScript):
grep -E "^import .* from " <path> | awk -F"from " '{print $2}' | sort -u | wc -l

# Same for Python:
grep -E "^(from |import )" <path> | sort -u | wc -l
```

**Instability:** `I = Ce / (Ca + Ce)` — compute in shell with `bc`:

```bash
echo "scale=2; $CE / ($CA + $CE)" | bc
```

**Compare against the last review.** Pull the previous Ca/Ce numbers from the previous foundationtik run's notes, or from a `last-instability` field if dockit records it. If neither exists, the rising-trend check can't fire on first run — the absolute threshold (`Ce > 5`) carries the signal alone.

**Limits to flag:**
- Grep-based fan-in misses re-exports through barrels and dynamic imports. For TypeScript, also grep for the foundation's named exports across the consumer glob.
- For Python, conditional imports inside function bodies are missed. They're rare in foundation consumers but worth noting.
- Aliased imports (`import foundation as f`) work fine for `from`-style detection but break `import` -style. Cover both forms.

## 6. Stale review — `foundation-stale-review`

**Read `last reviewed` from the registry row.** Format depends on dockit; assume ISO date or epoch.

**Check whether the path has been touched since:**

```bash
LAST_REVIEW="<iso-date or epoch>"
# Convert to epoch if needed:
LAST_EPOCH=$(date -d "$LAST_REVIEW" +%s)

# Most recent commit on the foundation:
LATEST_EPOCH=$(git log -1 --format=%ct -- <path>)

# Fires if older than 90 days AND code has changed since:
NINETY_DAYS_AGO=$(( $(date +%s) - 90*86400 ))
if [ "$LAST_EPOCH" -lt "$NINETY_DAYS_AGO" ] && [ "$LATEST_EPOCH" -gt "$LAST_EPOCH" ]; then
  echo "STALE"
fi
```

**Optional: list the commits in the staleness window for the ticket Evidence section:**

```bash
git log --since="@$LAST_EPOCH" --format='%h %ad %s' --date=short -- <path>
```

**Limits to flag:**
- High confidence: commit timestamps are authoritative. The only edge case is rebases that rewrite committer dates without changing author dates — use `%at` (author date) if your project rewrites history routinely.
- A foundation moved via `git mv` will lose its history unless `--follow` is used. Add `--follow` if the row's `path` looks recent.

## 7. Deprecation candidate — `foundation-deprecation-candidate`

Same fan-in computation as in section 5 (Ca):

```bash
grep -rl "from '<foundation-path>'" <consumer-glob> | wc -l
```

**Augment with re-export-aware search before recommending deletion.** Re-exports are the #1 way grep undercounts:

```bash
# Find barrels that re-export the foundation:
grep -rl "export .* from '<foundation-path>'" <consumer-glob>
# Then grep for usages of the re-exported names through those barrels:
# (project-specific — names depend on what the barrel re-exports)
```

**Trend (optional):** if dockit records historical Ca values, compare to the last review. A trending-down Ca + current Ca < 2 is the strongest deprecation signal. Without history, the absolute threshold alone fires the check (mark confidence accordingly).

**Limits to flag:**
- Dynamic imports, runtime-string `import()`s, and DI containers can hide consumers. **Always mark deprecation tickets as low-confidence** unless the agent has verified by other means (linter "unused export," IDE find-usages, etc.).
- Test files referencing the foundation count as consumers under naive grep — exclude `tests/`, `__tests__/`, `*.test.*`, `*.spec.*` from the fan-in if you want only production consumers.
- A foundation referenced only from the registry itself (`FOUNDATIONS.md` mentions it) is not a real consumer. The fan-in glob shouldn't include `*.md` anyway.

---

## How to wire heuristics into tickets

For every ticket, the Evidence section should:

1. **Quote the number** the heuristic produced (LOC, method count, Ca, Ce, instability, commit list).
2. **Quote the command** that produced it. Lets the reader reproduce.
3. **Flag confidence** — `Confidence: high` if the heuristic is reliable for this stack, `Confidence: low — <reason>` if the limits above apply.

Reproducible numbers + reproducible commands is the contract. A ticket that says "this is bloated" without a number gets ignored; one that says "LOC: 743 (`wc -l src/auth/index.ts`)" gets actioned.
