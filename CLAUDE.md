# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Tikkit is a ticket creation toolkit for AI agents. It turns various inputs into structured implementation tickets:
- Text requests → `tik`
- Figma designs → `figtik`
- Google Stitch UI exports → `stitchtik`
- Code-quality audits → `modernizer` (with the `auditor` agent)

All skills write to a shared `specs/` directory in the consuming project. Tikkit is a sibling of [repokit](https://github.com/TheLampshady/repokit), which handles documentation, onboarding, and code-quality checking.

There is no build system or compiled code. Everything is Markdown, TOML, and JSON.

## Directory Map

| Path | Purpose |
|------|---------|
| `skills/tik/` | Default ticket skill — turns text requests into tickets in `specs/tickets/` |
| `skills/figtik/` | Figma-to-ticket skill — fetches design data via API, creates implementation tickets |
| `skills/stitchtik/` | Stitch-to-ticket skill — analyzes Google Stitch UI exports against the codebase |
| `skills/modernizer/` | Code-modernization skill — audits tooling, writes tickets to `specs/` |
| `agents/auditor.agent.md` | Code/practice auditor — invoked by modernizer to find stale patterns |
| `.agents/skills/` | Symlink to `skills/` for Gemini cross-compatibility |
| `.claude-plugin/` | Claude plugin metadata (`plugin.json`) and marketplace catalog (`marketplace.json`) |
| `.mcp.json` | Bundled MCP servers (context7 for library documentation lookups) |
| `policies/` | Gemini CLI policy engine rules |
| `src/ticket-template.md` | Canonical ticket template — synced into tik/figtik/modernizer via `make sync` |
| `GEMINI.md` | Gemini extension context |
| `gemini-extension.json` | Gemini extension manifest |

## Architecture

### Skills (`skills/`, cross-platform)

Skills have YAML frontmatter (`name`, `description`, `user-invocable: true`) and load on demand. Claude discovers from `skills/` at plugin root; Gemini discovers from `.agents/skills/` (symlinked to `skills/`); Copilot discovers from `skills/` via plugin install.

| Skill | Modes | Key Behavior |
|-------|-------|-------------|
| `tik` | (single mode) | Default ticket skill — turns text requests into tickets in `specs/tickets/` with `[tik]` tag |
| `figtik` | create, update | Fetches Figma design data via API; compares against codebase; writes tickets with `[figtik]` tag |
| `stitchtik` | (single mode) | Analyzes Google Stitch UI exports (`screen.png`, `code.html`, `DESIGN.md`) against codebase; writes tickets with `[stitchtik]` tag |
| `modernizer` | analyze, status | Plans only, never executes; audits tooling and writes tickets with `[modernizer]` tag; uses `auditor` agent for stale-practice detection |

### Agents (`agents/`, distributed with plugin)

| Agent | Auto-triggers | Output |
|-------|-------------|--------|
| `auditor` | Reviews codebase for outdated code, stale practices, automation gaps | Returns findings report; does not write tickets directly — modernizer turns findings into tickets |

### Ticket System (`specs/`)

All tikkit skills write to a shared location in the consuming project:
- `specs/backlog.md` — master checklist, one line per item, tagged by source
- `specs/tickets/<slug>.md` or `specs/tickets/<slug>/ticket.md` — individual tickets with full context

Format in `backlog.md` — position in the list IS the priority/dependency order:
```
- [ ] Design system tokens [stitchtik] → tickets/design-system-tokens/ticket.md
- [ ] Bottom nav bar [stitchtik] → tickets/bottom-nav-bar/ticket.md
- [ ] Add dark mode support [tik] → tickets/dark-mode-support.md
- [ ] Hero section redesign [figtik] → tickets/hero-section-redesign/ticket.md
- [ ] Testing setup [modernizer] → tickets/testing-setup.md
```

All skills use plain kebab-case slugs — no numeric prefixes. Dependencies are expressed via position in the backlog and references inside each ticket.

Always check `specs/backlog.md` before creating a ticket to avoid duplicates.

### Cross-plugin contract with repokit

If the consuming project also has [repokit](https://github.com/TheLampshady/repokit) installed, both plugins share the same `specs/backlog.md`. Tag ownership:

| Tag | Owner |
|-----|-------|
| `[tik]`, `[figtik]`, `[stitchtik]`, `[modernizer]` | tikkit |
| `[sanity-checker]` | repokit |

The format is identical — neither plugin imports the other.

### Plugin Structure

This repo is both a **Claude plugin** and a **Gemini extension**:

- Claude: `.claude-plugin/plugin.json` (plugin metadata), `.claude-plugin/marketplace.json` (marketplace catalog pointing to `"./"`)
- Gemini: `gemini-extension.json` (extension manifest, references `GEMINI.md` as context file)

The root IS the plugin — there is no nested `plugins/` directory.

## Shared Templates

`src/ticket-template.md` is the canonical ticket structure. It is copied into `skills/tik/`, `skills/figtik/`, and `skills/modernizer/` via `make sync`. Edit `src/`, then run `make sync` before committing.

`stitchtik` is intentionally **not** a sync target — its template has Stitch-specific structure (Component Inventory, Design References, Responsive Requirements) that diverges from the canonical template. Update it manually when needed.

## Policies

`policies/policies.toml` applies to Gemini CLI only. Rules by category:

| Category | Rules |
|----------|-------|
| Destructive ops | Confirm `rm -rf`, confirm deleting `specs/` or `agents/` dirs |
| Git | Confirm `git push` |
| Secrets | Deny reading `.env`/`id_rsa`/`passwd`, deny writing to `.env*` |
| Context files | Confirm before overwriting `CLAUDE.md` or `GEMINI.md` |
| Safety checker | Path validation on all file writes |

## Development Commands

```bash
make setup      # First-time setup: install pre-commit hooks + link Gemini extension + install Claude plugin
make sync       # Copy src/ticket-template.md into tik, figtik, modernizer
make check      # Run all pre-commit validations (JSON, TOML, YAML)
make status     # Show open backlog items and extension link status
make help       # List all targets
```

`make hooks` uses `uv tool install` if uv is available, falls back to pip. Pre-commit config lives at `.config/.pre-commit-config.yaml`.

## Skill Frontmatter Format

```yaml
---
name: skill-name
description: 'Trigger phrases and what this skill does. Use when asked to: ...'
user-invocable: true
---
```

Keep `description` under 1024 characters. Include action verbs and "Use when asked to..." triggers.
