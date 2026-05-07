# tikkit

AI-powered ticket creation toolkit. Turns text requests, Figma designs, Stitch UI exports, code-quality audits, and `FOUNDATIONS.md` registry entries into structured implementation tickets — all landing in one shared backlog.

Sibling project: [repokit](https://github.com/TheLampshady/repokit) — documentation, onboarding, and code-quality checking.

---

## How It Works — Many Sources, One Backlog

Each skill takes a different input but writes to the same place. Position in the backlog is the priority/dependency order:

```
        Inputs                       Skills                      Shared output
        ──────                       ──────                      ─────────────

  ┌──────────────────┐         ┌──────────────┐
  │ Text request     │  ─────► │    /tik      │ ─────┐
  └──────────────────┘         └──────────────┘      │
                                                       │
  ┌──────────────────┐         ┌──────────────┐      │
  │ Figma URL        │  ─────► │   /figtik    │ ─────┤
  └──────────────────┘         └──────────────┘      │
                                                       │     ┌────────────────────┐
  ┌──────────────────┐         ┌──────────────┐      │     │ specs/backlog.md   │
  │ Stitch dir       │  ─────► │  /stitchtik  │ ─────┼───► │ specs/tickets/     │
  └──────────────────┘         └──────────────┘      │     └────────────────────┘
                                                       │
  ┌──────────────────┐         ┌──────────────┐      │
  │ FOUNDATIONS.md + │  ─────► │/foundationtik│ ─────┤
  │ codebase scan    │         └──────────────┘      │
  └──────────────────┘                                 │
                                                       │
  ┌──────────────────┐         ┌──────────────┐      │
  │ Codebase         │  ─────► │ /modernizer  │ ─────┘
  └──────────────────┘         └──────┬───────┘
                                       │ delegates audit to
                                       ▼
                                 ┌────────────┐         ┌──────────────────────┐
                                 │  auditor   │ ──uses──► context7 MCP        │
                                 │  (agent)   │         │ (live library docs)  │
                                 └────────────┘         └──────────────────────┘
```

---

## What's Inside

| Skill | Invoke | Use when... |
|-------|--------|-------------|
| `tik` | `/tik` | You have a **text request or bug report** with no Figma/Stitch context |
| `figtik` | `/figtik` | You have a **Figma URL** to ticket (or want to add Figma scope to an existing ticket) |
| `stitchtik` | `/stitchtik` | You have **Google Stitch exports** (`screen.png`, `code.html`, `DESIGN.md`) to ticket |
| `foundationtik` | `/foundationtik` | You have a **`FOUNDATIONS.md`** registry (from repokit's `dockit`) and want maintenance tickets for shared/foundational code — bloat, untested APIs, wrong-abstraction signals, coupling regressions, stale reviews, deprecation candidates |
| `modernizer` | `/modernizer` | You want a **codebase audit** for tooling, testing, or quality gaps |

| Agent | Purpose |
|-------|---------|
| `auditor` | Reviews codebase for stale practices and outdated tooling. Used internally by modernizer; queries context7 MCP to verify against current upstream docs. |

| MCP Server | Purpose |
|------------|---------|
| `context7` | Bundled HTTP MCP — fetches live documentation for libraries/frameworks so audits compare your code against the *current* upstream state, not training data. |

All output lands in `specs/backlog.md` (master checklist) and `specs/tickets/` (individual tickets) in the consuming project.

---

## Install

### Claude Code

```bash
/plugin marketplace add TheLampshady/tikkit
/plugin install tikkit@tikkit-marketplace
```

### Gemini CLI

```bash
gemini extensions install https://github.com/TheLampshady/tikkit
```

To enable the `auditor` subagent in Gemini, set `experimental.enableAgents: true` in `.gemini/settings.json` and copy `agents/*.md` into `.gemini/agents/`.

### Local development

```bash
git clone https://github.com/TheLampshady/tikkit
cd tikkit
make setup    # installs pre-commit hooks, links Gemini extension, installs Claude plugin
```

---

## Ticket Format

Backlog entries — position in the list IS the priority/dependency order:

```
- [ ] Design system tokens [stitchtik] → tickets/design-system-tokens/ticket.md
- [ ] Bottom nav bar       [stitchtik] → tickets/bottom-nav-bar/ticket.md
- [ ] Add dark mode        [tik]       → tickets/dark-mode-support.md
- [ ] Hero redesign        [figtik]    → tickets/hero-section-redesign/ticket.md
- [ ] Testing setup        [modernizer]→ tickets/testing-setup.md
```

Tags ship with tikkit: `[tik]`, `[figtik]`, `[stitchtik]`, `[foundationtik]`, `[modernizer]`. Slugs are plain kebab-case — no numeric prefixes.

Always check `specs/backlog.md` before creating a ticket to avoid duplicates.

If [repokit](https://github.com/TheLampshady/repokit) is also installed, it adds `[feedback-loop]` to the same file. The format is identical and neither plugin imports the other.

### `foundationtik` ↔ repokit cross-plugin contract

`foundationtik` is a tikkit skill but reads a repokit-generated artefact:

| Artefact | Owner plugin | Consumer |
|----------|--------------|----------|
| `FOUNDATIONS.md` | repokit (dockit generates, sync refreshes) | foundationtik reads (does NOT modify) |
| `specs/backlog.md` | shared | both plugins write |
| `specs/tickets/*.md` | shared | both plugins write |
| `[foundationtik]` tag | tikkit | this skill writes |

If `foundationtik` halts because `FOUNDATIONS.md` doesn't exist, run `/repokit:dockit` first. If it detects drift between the registry and the code, it files a `foundation-stale-review` ticket asking the user to run `/repokit:dockit sync` — it never edits the registry directly.

---

## Development

```bash
make help     # list targets
make sync     # copy src/ticket-template.md into tik, figtik, modernizer
make check    # validate JSON/TOML/YAML
make status   # open backlog items + install status
```

The canonical ticket template lives at `src/ticket-template.md` and is synced into `skills/tik/`, `skills/figtik/`, `skills/foundationtik/`, and `skills/modernizer/`. `stitchtik` has a custom template and is not a sync target.

---

## License

MIT
