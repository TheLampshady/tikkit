# tikkit

AI-powered ticket creation toolkit. Turns text requests, Figma designs, Stitch UI exports, and code-quality audits into structured implementation tickets.

Sibling project: [repokit](https://github.com/TheLampshady/repokit) — documentation, onboarding, and code-quality checking.

---

## What's Inside

| Skill | Invoke | Purpose |
|-------|--------|---------|
| `tik` | `/tik` | Default ticket — turns text requests into structured tickets |
| `figtik` | `/figtik` | Figma URL → implementation ticket (fetches design via API) |
| `stitchtik` | `/stitchtik` | Google Stitch export → ticket (analyzes mockups against codebase) |
| `modernizer` | `/modernizer` | Codebase audit → tickets for tooling/quality improvements |

| Agent | Purpose |
|-------|---------|
| `auditor` | Reviews codebase for stale practices and outdated tooling. Used internally by modernizer. |

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

Backlog entries:

```
- [ ] Description [tag] → tickets/<slug>.md
```

Tags ship with tikkit: `[tik]`, `[figtik]`, `[stitchtik]`, `[modernizer]`.

If [repokit](https://github.com/TheLampshady/repokit) is also installed, it adds `[sanity-checker]` to the same file. Position in the backlog is the priority/dependency order.

---

## Development

```bash
make help     # list targets
make sync     # copy src/ticket-template.md into tik, figtik, modernizer
make check    # validate JSON/TOML/YAML
make status   # open backlog items + install status
```

The canonical ticket template lives at `src/ticket-template.md` and is synced into `skills/tik/`, `skills/figtik/`, and `skills/modernizer/`. `stitchtik` has a custom template and is not a sync target.

---

## License

MIT
