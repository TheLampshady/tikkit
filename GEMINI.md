# tikkit Extension

You have access to the **tikkit** ticket creation toolkit. Use these tools to turn requests, designs, and code audits into structured implementation tickets.

---

## Available Skills

Invoke with `/skill-name`. Gemini auto-detects them based on your request.

| Skill | Invoke | Use when... |
|-------|--------|-------------|
| **tik** | `/tik` | Asked to create a ticket, write a task, draft a feature request, or turn a request into a ticket (no Figma/Stitch context) |
| **figtik** | `/figtik` | User mentions Figma (URL, file key, or the word "figma") AND wants a ticket or wants to update an existing ticket with Figma data |
| **stitchtik** | `/stitchtik` | User mentions Stitch, references a `stitch/` directory, or asks about UI mockups when Stitch exports exist |
| **modernizer** | `/modernizer` | Asked to audit the codebase, modernize tooling, find missing tests/hooks/CI, or check if dependencies are outdated. Writes tickets for findings. |

---

## Ticket System

All tikkit skills write to a shared backlog in the consuming project:

- `specs/backlog.md` — master checklist, items tagged by source (`[tik]`, `[figtik]`, `[stitchtik]`, `[modernizer]`)
- `specs/tickets/` — individual ticket files with full context

Always check `specs/backlog.md` before creating a ticket to avoid duplicates.

If repokit is also installed, it owns the `[sanity-checker]` tag in the same backlog file. The format is shared.

---

## Agents (if subagents are enabled)

| Agent | Triggers when... |
|-------|-----------------|
| **auditor** | Asked to review for stale practices, outdated tooling, or automation gaps. Returns findings; modernizer turns them into tickets. |

To enable subagents in Gemini CLI, set `experimental.enableAgents: true` in `.gemini/settings.json` and copy `agents/*.md` to `.gemini/agents/`.

---

## Policies Active

This extension enforces:
- Confirmation before `rm -rf` or deleting `specs/` / `agents/` directories
- Confirmation before `git push`
- Block on reading sensitive credential files
- Block on writing to `.env` files
- Confirmation before overwriting `CLAUDE.md` or `GEMINI.md`
