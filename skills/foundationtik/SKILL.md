---
name: foundationtik
description: 'Turn entries in FOUNDATIONS.md into maintenance tickets for shared/foundational code. Use when the user asks to: audit our foundations, ticket foundation work, refactor a foundation, address foundation bloat, deprecate a foundation, write tickets for shared code health, check if core/ is getting too big. Reads FOUNDATIONS.md (run /repokit:dockit if missing) and scans the code for god-class bloat (LOC + method count), untested public surface, wrong-abstraction signals (param/conditional growth — Sandi Metz), shotgun surgery across consumers, coupling regressions (afferent/efferent), stale reviews, and deprecation candidates. Writes tickets tagged [foundationtik]. Triggers: "audit our foundations", "create tickets for the core/ module", "is the auth foundation getting bloated", "what foundations need work", "deprecate the legacy session helper". Should NOT trigger for general ticket requests (use tik), Figma designs (figtik), Stitch exports (stitchtik), or dev-tooling modernization (modernizer).'
user-invocable: true
---

# foundationtik

Turn entries in a project's `FOUNDATIONS.md` registry into structured maintenance tickets for shared/foundational code.

Where the other tiks consume designs or text, foundationtik consumes **the foundations registry plus a code scan** and writes tickets describing health issues — bloat, untested APIs, wrong-abstraction signals, shotgun surgery, coupling regressions, stale reviews, and deprecation candidates.

## What This Skill Does (and Does Not)

- **Plans, never executes.** Writes tickets to `specs/tickets/`. Does not modify foundation code, tests, or `FOUNDATIONS.md` itself.
- **Reads, doesn't generate.** If `FOUNDATIONS.md` is missing, halts and points to `/repokit:dockit`. Documentation generation is dockit's job.
- **Uses native signals only.** No SonarQube, no custom metric collectors. Everything is computed with `git`, `grep`, `wc`, and friends — see `references/detection-heuristics.md`.

## Cross-Plugin Contract

`foundationtik` is a tikkit skill but reads a repokit-generated artefact:

| Artefact | Owner plugin | Consumer |
|----------|--------------|----------|
| `FOUNDATIONS.md` | repokit (dockit generates, sync refreshes) | foundationtik reads |
| `specs/backlog.md` | shared | both plugins write |
| `specs/tickets/*.md` | shared | both plugins write |
| `[foundationtik]` tag | tikkit | this skill writes |

If a foundation row appears stale (e.g. consumer count trending differently than the registry says), foundationtik writes a ticket recommending `/repokit:dockit sync` rather than editing `FOUNDATIONS.md` directly.

---

## Execution Flow

### Phase 1 — Pre-flight

1. **Locate `FOUNDATIONS.md`.** Search the project root, `docs/`, and `docs/architecture/`.
2. **If missing, halt:**
   > FOUNDATIONS.md not found. Run `/repokit:dockit` to generate it, then re-run this skill.

   Don't try to scan the codebase blind — without the registry, foundationtik has no opinion on what counts as "foundational."
3. **Parse the registry into rows.** Each row should expose at least: `name`, `type`, `path`, `owner`, `public API`, `consumers`, `depends-on`, `status`, `last reviewed`. Tolerate variations in column order or extra fields — the parser should be forgiving.
4. **Read `specs/backlog.md`.** Build a set of existing `[foundationtik]` tickets, keyed by `(foundation-slug, ticket-type-suffix)`. This is the duplicate-suppression set used in Phase 4.

### Phase 2 — Scope

Ask the user one question, then proceed:

> Which foundations? `all` (default) | `<foundation-name>` | a comma-separated list.

Default behaviour:
- `all` iterates over every active row.
- Skip rows with `status: deprecated` unless the user explicitly opts them in (e.g. they ask for tickets to *finish* deprecating something).

If the user names a foundation that isn't in the registry, list the available rows and ask which they meant.

### Phase 3 — Per-foundation scan

For each in-scope row, run the seven checks below. **Each check that fires produces one ticket.** See `references/detection-heuristics.md` for shell-only ways to compute each signal and the limits of each heuristic.

| Check | Signal | Ticket type |
|-------|--------|-------------|
| God class | LOC > 500 OR public-method count > 20 | `foundation-bloat` |
| Untested surface | Public method with no referencing test file | `foundation-untested-api` |
| Wrong abstraction | Parameter count or conditional count grew over last N commits on the foundation's primary file | `foundation-wrong-abstraction` |
| Shotgun surgery | A single commit touched ≥ 5 consumers on a foundation API change | `foundation-shotgun-surgery` |
| Coupling regression | Efferent coupling (Ce) > 5, OR instability `I = Ce/(Ca+Ce)` rising vs. last review | `foundation-coupling` |
| Stale review | `last reviewed` older than 90 days AND `git log` shows touches in that window | `foundation-stale-review` |
| Deprecation candidate | Active row with consumer count trending to zero (Ca < 2) | `foundation-deprecation-candidate` |

**Confidence flagging.** Some heuristics have known blind spots (e.g. grep-based fan-in misses re-exports, Python public-method detection misses `@property` and `__call__`). When a check fires through a low-confidence path, flag it in the ticket's Evidence section as `Confidence: low — <reason>` so the reader knows to verify before acting.

### Phase 4 — Write tickets

For each fired check:

1. **De-dupe.** Skip if a ticket with the same `(foundation, ticket_type)` pair already exists in `specs/backlog.md`. The point is to *grow* the backlog, not flood it with re-runs.
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

Append one line per ticket to `specs/backlog.md`. **Position is priority order.** Put the unblockers near the top:

1. `foundation-bloat` and `foundation-deprecation-candidate` first — they unblock other refactor work or remove it entirely.
2. `foundation-coupling` and `foundation-wrong-abstraction` next — they shape the surface other work depends on.
3. `foundation-untested-api`, `foundation-shotgun-surgery`, `foundation-stale-review` last — important but lower-leverage when sequenced against the others.

```
- [ ] Auth foundation bloat [foundationtik] → tickets/auth-foundation-bloat.md
- [ ] Cache deprecation candidate [foundationtik] → tickets/cache-deprecation-candidate.md
- [ ] Session helper coupling regression [foundationtik] → tickets/session-helper-coupling.md
- [ ] Auth foundation untested API [foundationtik] → tickets/auth-foundation-untested-api.md
```

If `specs/backlog.md` doesn't exist, create it with the entries.

### Phase 6 — Report

Summarise in plain language:

- Foundations scanned (and any skipped, e.g. deprecated rows)
- Tickets created (grouped by ticket type)
- Tickets skipped because they already exist
- Foundations that came up clean — these are good news, call them out

End the report with:

> Run `/repokit:repokit status` to see all open backlog items, or `/foundationtik` again after addressing some to refresh.

---

## Conventions

- **Default to "decide, don't ask."** Once the scope question is answered, run all seven checks against every in-scope foundation. Don't ask the user which checks to run — the registry plus the heuristics decide.
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
