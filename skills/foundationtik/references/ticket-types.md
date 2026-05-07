# foundationtik — Ticket Types

Seven ticket shapes, one per detection signal. Each shape extends the canonical tikkit ticket template (`./ticket-template.md`) — it does not replace it. Use the templates below to fill in the foundationtik-specific Goals sub-sections, Evidence content, Recommended Approach, and Acceptance Criteria.

The slug for every ticket is `<foundation-slug>-<ticket-type-suffix>`, e.g. `auth-foundation-bloat`, `cache-untested-api`.

## Index

| # | Ticket type | Trigger | Why it matters |
|---|-------------|---------|----------------|
| 1 | [`foundation-bloat`](#1-foundation-bloat) | registry-driven (`Health: hotspot` + size) | god class / file too large |
| 2 | [`foundation-untested-api`](#2-foundation-untested-api) | scan-driven | public surface without tests |
| 3 | [`foundation-wrong-abstraction`](#3-foundation-wrong-abstraction) | registry-driven (`Health: hotspot` + arity growth) | params/conditionals growing |
| 4 | [`foundation-shotgun-surgery`](#4-foundation-shotgun-surgery) | scan-driven | single change rippling across consumers |
| 5 | [`foundation-coupling`](#5-foundation-coupling) | scan-driven | efferent coupling or instability rising |
| 6 | [`foundation-stale-review`](#6-foundation-stale-review) | registry-driven (`Last Reviewed > 90d`) | registry review overdue |
| 7 | [`foundation-deprecation-candidate`](#7-foundation-deprecation-candidate) | registry-driven (`Consumers < 2`) | consumers trending to zero |

Registry-driven types pull their trigger from a field already in `FOUNDATIONS.md`; scan-driven types are computed by foundationtik against the foundation's `Path`. See `detection-heuristics.md` for the per-type recipes.

Each template uses these placeholders:
- `<Foundation>` — the human-readable foundation name (e.g. "Auth foundation")
- `<foundation-slug>` — kebab-case slug (e.g. `auth-foundation`)
- `<path>` — the foundation's primary file or directory
- Italics indicate text the agent should replace per-ticket; everything else is structural.

---

## 1. `foundation-bloat`

**Fires when:** Registry row has `Health: hotspot` AND the file clears the bloat threshold (LOC > 500 OR public-method/export count > 20).

**Why it matters:** A foundation that has grown into a god-object becomes the bottleneck the rest of the system can't refactor around. Splitting it lets parallel work resume.

```markdown
# <Foundation> bloat

## Overview

<Foundation> at `<path>` has grown to a size where it's hard to reason about as a single unit. The registry has flagged it as a hotspot. This ticket splits it into smaller, single-responsibility units so future work on this foundation can proceed in parallel without merge churn.

## Evidence

- Registry signal: `Health: hotspot` (FOUNDATIONS.md, last sync `<date>`)
- LOC: **<n>** (threshold: 500). Command: `wc -l <path>`
- Public methods/exports: **<n>** (threshold: 20). Command: `<heuristic from detection-heuristics.md>`
- Top-3 longest methods (LOC):
  - `<method-a>` — <n> lines (`<path>:<line>`)
  - `<method-b>` — <n> lines (`<path>:<line>`)
  - `<method-c>` — <n> lines (`<path>:<line>`)
- Confidence: <high | low — reason>

## Goals

* **Identify natural split lines**
  - Group public methods by collaborator/responsibility
  - Mark which methods would move to which new module
* **Extract sub-modules**
  - One module per cohesive group, named by responsibility (not by file split)
  - Update the foundation's public surface to re-export only what consumers need
* **Update consumers**
  - Migrate import paths for any moved symbols in the same PR
  - Update `FOUNDATIONS.md` (via `/repokit:dockit sync`) to reflect the new structure

## Recommended Approach

1. List the public surface and group by which other public method calls into it. Cohesive groups become candidate modules.
2. Extract one group at a time. After each extraction, run the test suite — green before moving on.
3. Re-export from the original path during migration so consumers don't churn until the final step.
4. Once all groups are extracted, drop the re-exports and update consumer imports in the final commit.

## Acceptance Criteria

* **Given** the foundation has been split into N sub-modules,
  **When** the test suite runs,
  **Then** all foundation tests pass with no regression.

* **Given** consumer code referencing the old path,
  **When** the rename PR merges,
  **Then** all consumers compile and their tests pass on the new path.

* **Given** the new structure,
  **When** `wc -l` is run on each new module,
  **Then** no module exceeds 500 LOC and no module has more than 20 public methods.

## Rollback

Single revert of the rename PR restores the original layout. Earlier extraction commits are behaviour-preserving and safe to keep if rollback is partial.
```

---

## 2. `foundation-untested-api`

**Fires when:** A public method/export has no referencing test file.

**Why it matters:** Untested foundation APIs are landmines. Every consumer trusts them; nothing verifies them. The fix is small but high-leverage.

```markdown
# <Foundation> untested API

## Overview

<Foundation> exposes public methods that no test file references. This ticket adds direct tests for the untested surface so consumers can rely on documented behaviour and refactors can move with confidence.

## Evidence

- Foundation path: `<path>`
- Untested public methods/exports:
  - `<method-a>` (`<path>:<line>`) — no matching test file under `tests/`
  - `<method-b>` (`<path>:<line>`) — no matching test file under `tests/`
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: <high | low — reason, e.g. "grep for method name may miss tests that exercise it indirectly via integration paths">

## Goals

* **Add direct unit tests for each untested public method**
  - Cover the documented behaviour, not the implementation
  - Include at least one happy-path and one boundary/error case per method
* **Document the contract**
  - If the public method's contract isn't obvious from the signature, add a docstring or update existing docs

## Recommended Approach

1. For each untested method, sketch the contract from its existing call sites: what inputs are passed, what outputs are consumed, what errors are caught.
2. Write tests against the contract, not the current implementation — they should keep passing if internals change.
3. Run coverage and confirm the new tests reach the targeted methods.

## Acceptance Criteria

* **Given** each previously untested public method,
  **When** the test suite runs,
  **Then** there is at least one test that directly exercises it.

* **Given** the new tests,
  **When** the test suite runs,
  **Then** all foundation tests pass with no regression in unrelated tests.

## Rollback

`git revert` of the test-addition commits. No production code changes, so rollback is trivial.
```

---

## 3. `foundation-wrong-abstraction`

**Fires when:** Registry row has `Health: hotspot` AND parameter count or conditional count on the foundation's primary file has grown over the last N commits (i.e. the hotspot signal is *not* explained by raw size — see `foundation-bloat` for that case).

**Why it matters:** Sandi Metz's observation: "duplication is far cheaper than the wrong abstraction." An abstraction accreting parameters and conditionals is leaking the differences it was supposed to hide. Inlining and re-extracting is usually cheaper than continuing to bend the existing shape.

```markdown
# <Foundation> wrong-abstraction signal

## Overview

<Foundation> at `<path>` is showing classic wrong-abstraction symptoms: the registry has flagged it as a hotspot, and callers keep adding parameters and conditionals to bend it to new use cases. This ticket inlines the abstraction back into its consumers and lets a better-fitting shape emerge.

> **Note:** This is a refactor with no behavioural change intended. If the codebase is mid-feature work that depends on this abstraction, sequence accordingly.

## Evidence

- Registry signal: `Health: hotspot` (FOUNDATIONS.md, last sync `<date>`)
- Foundation path: `<path>`
- Parameter count growth: from `<n_old>` to `<n_new>` over last <N> commits.
- Conditional growth: `<n_old>` → `<n_new>` `if`/`switch` statements over last <N> commits.
- Recent commits adding params/conditionals:
  - `<sha>` — `<commit subject>`
  - `<sha>` — `<commit subject>`
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: <high | low — reason, e.g. "diff-counting catches added if-statements but can't tell whether they were behaviour preserving or not">

## Goals

* **Inline the abstraction at every call site**
  - Replace the foundation function with the relevant branch's body at each consumer
  - Verify each consumer's tests stay green
* **Wait, then re-extract**
  - Resist the urge to immediately re-abstract. Live with the duplication for at least one feature cycle so the right seams become visible.
  - Once the actual shared structure is obvious from the duplication, extract a smaller, fitter abstraction.

## Recommended Approach

Follow the playbook in Sandi Metz, [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction):

1. Re-introduce the duplication. Inline the abstraction at every call site.
2. Re-run the tests after each inline; the goal is no behaviour change.
3. Defer re-extraction. The cure is duplication, not a different abstraction.
4. When a real pattern reasserts itself across the duplicated sites, extract a narrower abstraction that fits.

## Acceptance Criteria

* **Given** the abstraction has been inlined,
  **When** the test suite runs,
  **Then** all foundation tests and consumer tests pass with no regression.

* **Given** the inlined call sites,
  **When** code review compares before/after,
  **Then** behaviour is provably unchanged (same branches, same outputs).

* **Given** the inlined state,
  **When** future work begins,
  **Then** the team explicitly does NOT re-extract until at least one feature cycle has passed.

## Rollback

Single revert restores the abstraction. Inlining commits should be small and isolated to make this clean.
```

---

## 4. `foundation-shotgun-surgery`

**Fires when:** A single commit on a foundation API touched ≥ 5 consumers.

**Why it matters:** If one change forces edits across many consumers, the foundation's surface is leaking implementation. Either the API is too granular, or consumers are doing work the foundation should own.

```markdown
# <Foundation> shotgun surgery

## Overview

A recent change to <Foundation>'s public API rippled across <N> consumers in a single commit. This ticket pulls the duplicated change-driver into the foundation itself so future updates land in one place.

## Evidence

- Foundation path: `<path>`
- Triggering commit: `<sha>` — "<commit subject>"
- Consumers touched in the same commit:
  - `<consumer-path-1>:<line>`
  - `<consumer-path-2>:<line>`
  - `<consumer-path-3>:<line>`
  - `<consumer-path-4>:<line>`
  - `<consumer-path-5>:<line>`
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: <high | low — reason>

## Goals

* **Identify the leaked responsibility**
  - What logic did every consumer have to add/change in the same way?
  - Decide whether it belongs in the foundation, in a new layer between, or as a default the foundation provides
* **Pull the duplication into the foundation**
  - Add the behaviour or the missing default
  - Remove the duplicated handling from each consumer

## Recommended Approach

1. Diff the consumer changes from the triggering commit. The common shape is the leaked responsibility.
2. Decide where it belongs: foundation method, new helper layer, sensible default in the existing API.
3. Implement once in the foundation; remove the duplicated handling from consumers.
4. Add a regression test that exercises the new foundation behaviour from a representative consumer scenario.

## Acceptance Criteria

* **Given** the foundation has absorbed the previously duplicated logic,
  **When** the test suite runs,
  **Then** consumer behaviour is unchanged and all tests pass.

* **Given** a future change of the same kind,
  **When** the team makes it,
  **Then** the change lands in one place (the foundation), not <N>.

## Rollback

Revert the foundation change and the consumer simplifications together. They were introduced as a unit and back out as a unit.
```

---

## 5. `foundation-coupling`

**Fires when:** Efferent coupling `Ce > 5` OR instability `I = Ce / (Ca + Ce)` rising versus the last review.

**Why it matters:** A foundation should be *stable* (low instability) — many things depend on it, and it depends on few things. When `I` rises, the foundation is becoming a client of the rest of the system, which inverts its role.

```markdown
# <Foundation> coupling regression

## Overview

<Foundation> at `<path>` has gained outgoing dependencies. As a foundation it should be **stable** (depended on by many, depending on few). This ticket trims the new outgoing dependencies and restores the inversion.

## Evidence

- Foundation path: `<path>`
- Afferent coupling Ca (incoming): <n>
- Efferent coupling Ce (outgoing): <n> (threshold: 5)
- Instability I = Ce / (Ca + Ce): <value> (last review: <prior value>)
- New outgoing dependencies since last review:
  - `<module-a>` (introduced in `<sha>`)
  - `<module-b>` (introduced in `<sha>`)
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: <high | low — reason, e.g. "import-graph derived from grep misses dynamic imports">

## Goals

* **Justify or remove each new outgoing dependency**
  - For each `<module-X>`: is it strictly needed by foundation logic, or did it leak in?
  - Where it leaked in, invert the dependency (move the calling code to the consumer side)
* **Restore stable instability**
  - Target Ce ≤ 5 and I trending down vs. last review

## Recommended Approach

Follow the dependency-inversion principle (Robert C. Martin, *Clean Architecture*):

1. List each outgoing dependency. For each, ask: *does the foundation truly need this, or is it doing work that belongs in the caller?*
2. For each dependency that leaked in, push the calling code outward — the consumer can pre-compute the value and pass it in, or implement an interface the foundation declares.
3. Re-measure Ce and I after the changes. If `I` is still rising, repeat.

## Acceptance Criteria

* **Given** the foundation's outgoing dependency list,
  **When** the foundation tests run,
  **Then** they pass with no regression.

* **Given** the post-refactor measurement,
  **When** Ce and I are recalculated,
  **Then** Ce ≤ 5 and I is lower than at the last review.

* **Given** consumer integrations,
  **When** the consumers run their tests,
  **Then** behaviour is unchanged (consumers may now own logic that previously lived in the foundation).

## Rollback

Revert the foundation refactor and the consumer follow-ons together. Coupling changes touch both sides.
```

---

## 6. `foundation-stale-review`

**Fires when:** `Last Reviewed` in the registry row is older than 90 days AND `git log` shows touches to the foundation path within that window.

**Why it matters:** The registry says "we last looked at this on date X." If the code has changed since then, the registry's claims about public API, consumers, and dependencies may be lying. A fresh review either confirms the registry or surfaces what changed.

```markdown
# <Foundation> stale review

## Overview

<Foundation>'s registry entry was last reviewed <N> days ago, but the code has been touched in that window. This ticket walks the foundation against its registry entry, confirms or corrects the public API / consumers / dependencies, and re-stamps the review date.

## Evidence

- Registry signal: `Last Reviewed` field (FOUNDATIONS.md row for `<Foundation>`)
- Foundation path: `<path>`
- Last reviewed (per `FOUNDATIONS.md`): `<date>` (<N> days ago, threshold 90)
- Commits touching `<path>` since last review:
  - `<sha>` — "<subject>" (<date>)
  - `<sha>` — "<subject>" (<date>)
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: high (commit dates are authoritative)

## Goals

* **Walk the registry entry against the current code**
  - Public API: does the listed surface still match what's exported?
  - Consumers: rerun fan-in, update the count and list
  - Depends-on: rerun the import scan, update the list
* **Re-stamp the review**
  - Update `last reviewed` to today (via `/repokit:dockit sync`)
* **If the API changed materially, file a follow-up ticket**
  - Don't try to refactor in this ticket — note the drift and ticket it separately

## Recommended Approach

1. Read the current state of the foundation: public surface, imports, consumers via fan-in heuristic.
2. Compare line-by-line against the registry row. Flag every mismatch.
3. Run `/repokit:dockit sync` to update the row, or hand-edit the row if dockit doesn't cover the field.
4. If consumer count or public API has shifted in ways that suggest other foundationtik checks would now fire (e.g. coupling regression), re-run `/foundationtik` after the sync to pick them up.

## Acceptance Criteria

* **Given** a re-stamped registry row,
  **When** the row is read,
  **Then** `last reviewed` is today and `public API`, `consumers`, `depends-on` match the code.

* **Given** material drift between the old row and the code,
  **When** the review completes,
  **Then** any drift that warrants its own work is filed as a follow-up ticket (could be `foundation-bloat`, `foundation-coupling`, etc.).

## Rollback

This ticket is metadata-only (registry edits and follow-up tickets). Rollback is `git revert` of the dockit sync commit.
```

---

## 7. `foundation-deprecation-candidate`

**Fires when:** Registry row has `Status: active` AND `Consumers` count < 2.

**Why it matters:** A foundation with one or zero consumers is no longer a foundation. Either inline it into the remaining consumer or remove it. Either way, the registry should stop listing it as active.

```markdown
# <Foundation> deprecation candidate

## Overview

<Foundation> at `<path>` is listed as active in the registry but has fewer than 2 consumers. This ticket either inlines it into the remaining consumer or removes it outright, and updates the registry to reflect the decision.

## Evidence

- Registry signal: `Status: active`, `Consumers: <n>` (FOUNDATIONS.md row for `<Foundation>`)
- Foundation path: `<path>`
- Afferent coupling Ca (consumers): <n> (threshold for deprecation: < 2)
- Current consumers:
  - `<consumer-path-1>:<line>` (or "none")
- Registry status: `active`
- Detection command: `<heuristic from detection-heuristics.md>`
- Confidence: <high | low — reason, e.g. "fan-in via grep misses re-exports — verify before deletion">

## Goals

* **Decide: inline, deprecate, or keep**
  - **Inline** — if there's exactly one consumer and the foundation is small, move the code into the consumer.
  - **Deprecate** — if there are zero consumers, mark the row `deprecated` in the registry and schedule deletion.
  - **Keep** — only if there's a documented reason a future consumer needs it.
* **Execute the chosen path**
  - Inline path: move the code, delete the foundation file, update the consumer's tests.
  - Deprecate path: change `status` in the registry, add a deprecation notice in the foundation file, schedule deletion in N days.

## Recommended Approach

1. Verify the consumer count is real — re-run fan-in including re-export-aware searches before deleting anything (see `detection-heuristics.md` for the gotchas).
2. Talk to the row owner if listed. They may know about a planned consumer that justifies keeping it.
3. Choose the smallest reversible step: deprecation is reversible, deletion isn't. Default to deprecation first if you're not sure.
4. Always update the registry in the same PR as the code change.

## Acceptance Criteria

* **Given** the chosen path is "inline",
  **When** the PR merges,
  **Then** the foundation file is gone, the consumer's tests pass, and the registry row is removed.

* **Given** the chosen path is "deprecate",
  **When** the PR merges,
  **Then** the registry row's `status` is `deprecated`, the foundation file has a deprecation notice, and a follow-up ticket is filed for deletion.

* **Given** the chosen path is "keep",
  **When** the PR merges,
  **Then** the registry row has a comment explaining why it's kept and which future consumer is expected.

## Rollback

- Inline path: `git revert` restores the foundation file and the consumer's pre-inline state.
- Deprecate path: `git revert` flips the registry row back to active.
- Keep path: nothing to roll back.
```
