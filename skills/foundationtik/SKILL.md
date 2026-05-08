---
name: foundationtik
description: 'Turn entries in FOUNDATIONS.md into maintenance tickets for shared/foundational code. Use when the user asks to: audit our foundations, ticket foundation work, refactor a foundation, address foundation bloat, deprecate a foundation, write tickets for shared code health, check if core/ is getting too big. Reads FOUNDATIONS.md (run /repokit:dockit if missing) and scans the code for god-class bloat (LOC + method count), untested public surface, wrong-abstraction signals (param/conditional growth — Sandi Metz), shotgun surgery across consumers, coupling regressions (afferent/efferent), stale reviews, and deprecation candidates. Writes tickets tagged [foundationtik]. Triggers: "audit our foundations", "create tickets for the core/ module", "is the auth foundation getting bloated", "what foundations need work", "deprecate the legacy session helper". Should NOT trigger for general ticket requests (use tik), Figma designs (figtik), Stitch exports (stitchtik), or dev-tooling modernization (modernizer).'
user-invocable: true
---

# foundationtik

Turn entries in a project's `FOUNDATIONS.md` registry into structured maintenance tickets for shared/foundational code.

Where the other tiks consume designs or text, foundationtik consumes **the foundations registry plus a code scan** and writes tickets describing health issues — bloat, untested APIs, wrong-abstraction signals, shotgun surgery, coupling regressions, stale reviews, and deprecation candidates.

## What This Skill Does (and Does Not)

- **Plans, never executes.** Writes tickets to `.backlog/tickets/`. Does not modify foundation code, tests, or `FOUNDATIONS.md` itself.
- **Reads, doesn't generate.** Documentation generation is dockit's job. If `FOUNDATIONS.md` is missing, foundationtik recommends `/repokit:dockit` for the full picture and offers an ad-hoc fallback (scan-driven checks only) for users who can name foundation paths themselves — see Phase 1.
- **Uses native signals only.** No SonarQube, no custom metric collectors. Everything is computed with `git`, `grep`, `wc`, and friends — see `references/detection-heuristics.md`.

## Cross-Plugin Contract

`foundationtik` is a tikkit skill but reads a repokit-generated artefact:

| Artefact | Owner plugin | Consumer |
|----------|--------------|----------|
| `FOUNDATIONS.md` | repokit (dockit generates, sync refreshes) | foundationtik reads |
| `.backlog/backlog.md` | shared | both plugins write |
| `.backlog/tickets/*.md` | shared | both plugins write |
| `[foundationtik]` tag | tikkit | this skill writes |

If foundationtik detects drift between the registry and the code (e.g. `Last Reviewed` > 90 days with commits in that window), it writes a `foundation-stale-review` ticket recommending `/repokit:dockit sync` rather than editing `FOUNDATIONS.md` directly.

---

## Execution Flow

### Phase 1 — Pre-flight

1. **Locate `FOUNDATIONS.md`.** Search the project root, `docs/`, and `docs/architecture/`.
2. **If missing**, give the user two options:

   > FOUNDATIONS.md not found. Two options:
   > 1. **Recommended:** run `/repokit:dockit` to generate the registry, then re-run me. You'll get all seven check types, priority ranking by `foundation_score`, and the Findings section.
   > 2. **Ad-hoc mode:** name the files/dirs you want treated as foundations and I'll run only the **scan-driven** checks against them (untested-api, shotgun-surgery, coupling). The registry-driven four (bloat, wrong-abstraction, stale-review, deprecation) need fields dockit produces and will be skipped.

   If the user picks option 2, ask once: *"Which paths? (e.g. `src/auth/`, `packages/core/src/index.ts`, comma-separated)"*. Validate each path exists; if not, list the project's top-level dirs and ask again. Then **skip step 3** below — there's no registry to parse — and proceed to step 4. In Phase 3, treat each user-named path as a synthetic row with `Status: active`, `Health: unknown`, no `Last Reviewed`, no `Consumers`. Don't fabricate the missing fields.
3. **Parse the registry.** *(Skip in ad-hoc mode.)* `FOUNDATIONS.md` is a dockit-generated catalog with three sections:
   - **Catalog table** — one row per foundation: `Name | Type | Path | Owner | Status | Health | Consumers | Last Reviewed`. **Row order is the ranking.** dockit sorts by `foundation_score` (fan-in × cross-feature × stability), so the top of the table is the most-foundational code. foundationtik preserves that order when writing tickets to the backlog.
   - **Per-foundation deep-dive sections** — one per row, with: Purpose, Public API, Invariants, Consumers table, Dependencies, Test coverage, Refactor triggers, Change checklist.
   - **Findings section** — separate from the catalog. Lists Hotspots, Hidden foundations, Pretenders. Hidden foundations and pretenders are **not foundationtik's responsibility** — dockit and the human reviewer handle them. Hotspots are also marked on the catalog row with `health: hotspot`, so foundationtik picks them up there.

   Field values to know:

   | Field | Values | Notes |
   |-------|--------|-------|
   | `Status` | `active` · `experimental` · `deprecated` · `sunset` | Default scope = `active` (and `experimental` on opt-in). |
   | `Health` | `healthy` · `hotspot` · `unknown` | `hotspot` is dockit's pre-classification of a refactor target. `unknown` = low-confidence detection — propagate to derived tickets. |
   | `Type` | `service` · `abstraction` · `primitive` · `design-system` | Used for ticket framing only. |

4. **Read `.backlog/backlog.md`.** Build a set of existing `[foundationtik]` tickets keyed by `(foundation-slug, ticket-type-suffix)` for duplicate suppression in Phase 4.

### Phase 2 — Scope

Ask the user one question, then proceed:

> Which foundations? `all active` (default) · `hotspots only` · `<foundation-name>` · a comma-separated list.

Default behaviour:
- `all active` iterates over rows with `status: active`.
- `hotspots only` filters to `health: hotspot` rows — the highest-leverage subset.
- Skip `status: deprecated` rows unless the user opts them in (e.g. they want tickets to *finish* a deprecation).
- If the user names a foundation that isn't in the registry, list the available rows and ask.

### Phase 3 — Per-foundation scan

For each in-scope row, the seven checks split into two groups: **registry-driven** (the trigger is already a field in `FOUNDATIONS.md` — no re-scan needed to *fire* the check) and **scan-driven** (foundationtik computes the signal because the registry doesn't carry it).

In both groups, the **Evidence section of each ticket should still cite concrete numbers** — registry fields tell you whether to fire, but the ticket reader wants to see the raw measurements (LOC, untested method names, commit hashes). The heuristics in `references/detection-heuristics.md` produce those numbers.

**In ad-hoc mode** (no FOUNDATIONS.md), only scan-driven checks run. Every ticket carries `Confidence: low — ad-hoc mode without registry` so the reader knows the inputs weren't validated against a registry.

#### Registry-driven (trigger from FOUNDATIONS.md)

| Trigger | Ticket type |
|---------|-------------|
| `Health: hotspot` AND file is bloated (LOC > 500 OR > 20 public methods) | `foundation-bloat` |
| `Health: hotspot` AND parameter/conditional growth dominates | `foundation-wrong-abstraction` |
| `Last Reviewed` > 90 days AND `git log` shows touches since | `foundation-stale-review` |
| `Consumers` count < 2 on an active row | `foundation-deprecation-candidate` |

When `Health: hotspot` fires, foundationtik picks **bloat or wrong-abstraction** by looking at the file:
- If the LOC/method-count threshold is exceeded, it's bloat.
- Otherwise it's wrong-abstraction.
- If both apply (large *and* growing in arity), write the bloat ticket — splitting the file unblocks the wrong-abstraction follow-up — and note the follow-up in the bloat ticket's Goals.

A `Health: hotspot` row that *doesn't* clear either heuristic is unusual but possible (a small file with fast churn for non-arity reasons). In that case, default to `foundation-wrong-abstraction` and mark `Confidence: low — hotspot classification did not match standard signatures, recommend manual review`.

#### Scan-driven (foundationtik computes)

| Check | Signal | Ticket type |
|-------|--------|-------------|
| Untested surface | Public method/export with no referencing test file | `foundation-untested-api` |
| Shotgun surgery | A single commit on the foundation's API touched ≥ 5 consumers | `foundation-shotgun-surgery` |
| Coupling regression | Efferent coupling Ce > 5, or instability `I = Ce/(Ca+Ce)` rising vs. last review | `foundation-coupling` |

The registry doesn't carry these signals — run the heuristics from `references/detection-heuristics.md` against the foundation's `Path`. Use the row's `Consumers` count as `Ca` to save a recompute when calculating instability.

#### Confidence

- Row `Health: unknown` → every derived ticket inherits `Confidence: low — registry detection was low-confidence`.
- Heuristic-with-blind-spots → flag `Confidence: low — <specific reason>` in the ticket's Evidence section. The blind spots are documented per heuristic in `references/detection-heuristics.md`.

#### What foundationtik does *not* do

- **Hidden foundations** (Findings section, "high fan-in but wrong directory") — dockit flags these and the human relocates. foundationtik does not write a ticket for relocation.
- **Pretenders** (low fan-in but living in `core/`/`shared/`/`lib/`) — out-of-band findings, never registry rows. foundationtik never sees them.

### Phase 4 — Write tickets

For each fired check:

1. **De-dupe.** Skip if a ticket with the same `(foundation, ticket_type)` pair already exists in `.backlog/backlog.md`. The point is to *grow* the backlog, not flood it with re-runs.
2. **Slug.** Use `<foundation-slug>-<ticket-type-suffix>`:
   - `auth-foundation-bloat`
   - `cache-untested-api`
   - `session-helper-deprecation-candidate`
3. **Template.** Use the canonical tikkit ticket template at `./references/ticket-template.md` (kept in sync via `make sync`). foundationtik adds a few mandatory sub-sections; full templates per ticket type are in `references/ticket-types.md`.

Required sections, in order:

- **Overview** — which foundation, which signal, plain-language description of the problem. Written for a tech lead, not a metrics dashboard.
- **Evidence** — concrete numbers (LOC, method names, commit hashes, consumer paths). Cite at least one `file:line`. Include the heuristic command that produced the number so a reader can reproduce it.
- **Goals** — what "fixed" looks like. Specific to the ticket type (see `references/ticket-types.md`).
- **Recommended approach** — language- and ticket-type-specific. For `foundation-wrong-abstraction`, reference Sandi Metz's [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) — the cure for a bad abstraction is duplication, not more abstraction.
- **Acceptance criteria** — Given/When/Then. Always include:
  - "No regression in foundation tests"
  - "Consumers updated in the same PR if the foundation's public API changed"
- **Rollback** — git revert plan. Foundations are load-bearing; every ticket must spell out how to back out cleanly.

### Phase 5 — Update backlog

Append one line per ticket to `.backlog/backlog.md`. **Priority order mirrors registry order.** Foundations near the top of the FOUNDATIONS.md catalog have the highest `foundation_score` — they're the most leverage to fix. Walk the catalog top-to-bottom and emit tickets per foundation in this order:

1. **Hotspot tickets first** (`foundation-bloat` or `foundation-wrong-abstraction`) — dockit pre-flagged them as the refactor target.
2. **Deprecation** (`foundation-deprecation-candidate`) — finishing a deprecation removes work.
3. **Stale review** — fast and frees later runs from re-firing the same trigger.
4. **Scan-driven tickets** (`foundation-untested-api`, `foundation-coupling`, `foundation-shotgun-surgery`) — important, lower-leverage when sequenced against the above.

```
- [ ] Auth foundation hotspot (bloat) [foundationtik] → tickets/auth-foundation-bloat.md
- [ ] Auth foundation untested API [foundationtik] → tickets/auth-foundation-untested-api.md
- [ ] Cache deprecation candidate [foundationtik] → tickets/cache-deprecation-candidate.md
- [ ] Helpers (hidden) coupling regression [foundationtik] → tickets/helpers-coupling.md
```

If `.backlog/backlog.md` doesn't exist, create it with the entries.

### Phase 6 — Report

Summarise in plain language:

- Foundations scanned (with their registry `Status` and `Health`)
- Tickets created — group by ticket type and call out which were triggered by registry fields vs. computed signals
- Tickets skipped because they already exist
- Foundations that came up clean — call them out, they're good news
- Findings the registry surfaces but foundationtik does not act on (Hidden foundations, Pretenders) — point the user back to dockit

End the report with:

> Run `/repokit:repokit status` to see all open backlog items, or `/foundationtik` again after addressing some to refresh.

If foundationtik ran in ad-hoc mode, also append:

> For the full check set (bloat, wrong-abstraction, stale-review, deprecation), run `/repokit:dockit` to generate FOUNDATIONS.md and then re-run me — those four need registry fields dockit produces.

---

## Conventions

- **Default to "decide, don't ask."** Once the scope question is answered, run every applicable check against every in-scope foundation — all seven with a registry, the three scan-driven ones in ad-hoc mode. Don't ask the user which checks to run; the registry plus the heuristics decide.
- **Prefer evidence to opinion.** Tickets must cite real numbers and real `file:line` references. If a check fires but can't produce concrete evidence, drop the ticket rather than ship a vague one.
- **Reference essays and posts, not paraphrases.** When recommending an approach (Sandi Metz on wrong abstractions, Robert Martin on coupling/instability), link to the source. The reader should be able to follow up.
- **Stay non-destructive.** No edits to foundation code, tests, or the registry itself.
- **Slugs are kebab-case.** `auth-foundation-bloat`, not `001-auth-bloat`. Position in the backlog encodes priority.
- **Reuse the canonical template.** foundationtik is a `make sync` target. Edit `src/ticket-template.md` upstream and re-sync — don't drift the local copy.

---

## References

- `references/ticket-template.md` — canonical tikkit ticket template (synced from `src/ticket-template.md`).
- `references/ticket-types.md` — full templates and worked examples for each of the seven foundationtik ticket shapes.
- `references/detection-heuristics.md` — shell-only methods for computing each signal, with the limits of each heuristic so the agent can flag low-confidence detections.
- Sandi Metz, [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) — cited from `foundation-wrong-abstraction` tickets.
- Robert C. Martin, *Clean Architecture* — afferent/efferent coupling and the instability metric `I = Ce/(Ca+Ce)`, cited from `foundation-coupling` tickets.
