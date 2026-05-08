# modernizer

Analyze codebases for AI-readiness and generate actionable plans. This skill is the **planning brain** — it knows best practices across languages, discovers available executors, and creates structured ticket files that agents and skills can consume.

modernizer is one of four skills in the [tikkit](../../README.md) toolkit. All four skills (`tik`, `figtik`, `stitchtik`, `modernizer`) write to the same `.backlog/backlog.md` so codebase audit findings live alongside text-, Figma-, and Stitch-sourced tickets.

## Philosophy

- **modernizer plans, others execute** — Focuses on analysis and recommendations
- **Language agnostic** — Supports Python, JS/TS, Java, Go, Rust, and more
- **Plans are machine-readable** — Output structured tickets for agents
- **Technology expert** — Knows latest best practices per language (2025)
- **Discovery-driven** — Finds available agents/skills and recommends which to use

## Installation

modernizer ships as part of the **tikkit** plugin/extension — there's no need to copy this folder into your project manually.

### Claude Code

```bash
/plugin marketplace add TheLampshady/tikkit
/plugin install tikkit@tikkit-marketplace
```

### Gemini CLI

```bash
gemini extensions install https://github.com/TheLampshady/tikkit
```

To enable the `auditor` subagent (used internally by modernizer for freshness audits), set `experimental.enableAgents: true` in `.gemini/settings.json` and copy `agents/*.md` into `.gemini/agents/`.

## Usage

```
/modernizer          # Analyze codebase, generate tickets in .backlog/
/modernizer status   # Check status, clean up completed tickets
```

### Analyze Mode (default)
Analyzes the codebase and generates:
- **Conversation**: Detailed recommendations tailored to your preferences
- **`.backlog/CHECKLIST.md`**: Scorecard for tracking progress
- **`.backlog/backlog.md`**: Append one line per ticket, tagged `[modernizer]`
- **`.backlog/tickets/<slug>.md`**: Individual ticket files for agents and downstream SDD frameworks (e.g., speckit)

### Status Mode
Shows ticket progress and cleans up:
- Reports completed vs remaining tickets
- **Deletes completed ticket files** from `.backlog/tickets/`
- Updates `.backlog/CHECKLIST.md`
- Shows next priority ticket

## Output Structure

```
.backlog/
├── backlog.md                       # Master checklist — shared with tik/figtik/stitchtik
├── CHECKLIST.md                     # modernizer's scorecard + ticket overview
└── tickets/                         # Individual ticket files
    ├── testing-setup.md
    ├── package-modernization.md
    └── ...
```

Backlog entries land alongside any tickets created by sibling tikkit skills:

```
- [ ] Testing setup            [modernizer] → tickets/testing-setup.md
- [ ] Package modernization    [modernizer] → tickets/package-modernization.md
- [ ] Hero section redesign    [figtik]     → tickets/hero-section-redesign/ticket.md
- [ ] Design system tokens     [stitchtik]  → tickets/design-system-tokens/ticket.md
```

Slugs are plain kebab-case — no numeric prefixes. Position in `backlog.md` IS the priority/dependency order.

Detailed recommendations are given directly in conversation. Language references live in `references/languages/*.md`.

## Supported Languages

| Language | Package Manager | Linter/Format | Testing |
|----------|-----------------|---------------|---------|
| Python | uv | ruff + ty | pytest |
| JS/TS (Frontend) | pnpm | ESLint / Biome | Vitest |
| JS/TS (Backend) | pnpm (Node) | ESLint / Biome | Vitest |
| Java/Kotlin | Gradle 8+ | Spotless | JUnit 5 |
| Go | go mod | golangci-lint | go test |
| Rust | cargo | clippy + rustfmt | built-in |

Each language has detailed recommendations in `references/languages/*.md` with:
- **Established vs Modern** options for each tool category
- **Licensing information** for compliance checks
- **Bun vs Node.js guidance** for JS/TS backends
- CLI framework recommendations
- Project structure examples

## What It Checks

| Category | Checks | Language-Specific |
|----------|--------|-------------------|
| **Documentation** | README, AI instructions, architecture | Universal |
| **Package Management** | Manager, lockfile, deps | Per-language best practice |
| **Testing** | Framework, coverage, isolation | Per-language tooling |
| **Code Quality** | Linting, formatting, types | Per-language tooling |
| **Code Patterns** | Hints, style, organization | Idiomatic per language |
| **CLI Frameworks** | CLI library choice (if applicable) | Typer, oclif, Cobra, clap, picocli |

## Ticket File Format

Each ticket follows the canonical tikkit ticket template (`./references/templates/ticket-template.md`) and is structured for both human review and agent consumption.

modernizer extends the base template with execution metadata under **Goals**:

```markdown
# <Title>

## Overview
What's out of date and what improves when this is done.

## Goals

* Specific goal bullets...

* **Current State** — What currently exists (files, configs, behavior)
* **Desired State** — What should exist when complete
* **Execution**
  - **Priority**: P1 | P2 | P3
  - **Category**: testing | packaging | linting | documentation | structure
  - **Language**: Python | JavaScript | TypeScript | Java | Go | Rust | Multi
  - **Executor**: [AGENT_NAME] | [SKILL_NAME] | manual
  - **Depends On**: [TICKET_SLUGS] or none
  - **Status**: pending | in_progress | completed
* **Implementation Notes** — Files to modify, recommended approach
* **Verification** — Commands + expected output
* **Rollback** — How to revert

## Acceptance Criteria

- **Given** ... **When** ... **Then** ...
- All existing tests pass
- No regressions introduced

## Tech Details
- Feature: <NAME>
- Type: chore | enhancement | bugfix
- Labels: ai-readiness, tooling, testing, ...
```

## Workflow

### Analyze (`/modernizer`)

```
/modernizer
    │
    ├── 1. Discovery
    │   ├── Detect language(s)
    │   ├── Find AI instruction files
    │   ├── Discover available agents/skills
    │   └── Detect project structure
    │
    ├── 1.5. User Preferences (interactive)
    │   ├── Modern vs Established tools?
    │   ├── Licensing requirements?
    │   └── JS/TS: Frontend or Backend?
    │
    ├── 2. Analysis (using language references)
    │   ├── Score: docs, packages, tests, quality, patterns
    │   └── (optional) Delegate freshness check to auditor agent ──► context7 MCP
    │
    ├── 3. Plan Generation
    │   ├── .backlog/CHECKLIST.md
    │   ├── .backlog/backlog.md   (append [modernizer] entries)
    │   └── .backlog/tickets/<slug>.md
    │
    └── 4. Discussion
        ├── Present summary
        ├── Ask clarifying questions
        └── Offer to run available executors
```

### Status (`/modernizer status`)

```
/modernizer status
    │
    ├── Read ticket files in .backlog/tickets/
    ├── Cross-reference with code state (acceptance criteria met?)
    ├── Delete completed tickets (status: completed)
    ├── Update .backlog/CHECKLIST.md
    └── Report: completed, remaining, next priority
```

## Integration with Executors

modernizer discovers available agents and skills:

```
Available Executors:
- Agent: auditor          → Doc/dependency freshness audits (built in)
- Agent: test-scaffolder  → Testing setup tickets
- Agent: feedback-loop    → Code quality verification
- Skill: dockit           → Documentation generation
- Skill: speckit          → SDD framework (spec/ticket workflow)
```

Then matches tickets to executors and offers to run them:

```
Testing ticket detected:
  → test-scaffolder agent available
    "Run test-scaffolder for testing-setup.md?"
```

## SDD Framework Integration

Ticket files are structured for SDD framework compatibility (speckit by default) — the `Goals` sub-sections, Acceptance Criteria, and Tech Details map cleanly onto a framework ticket. If `.specify/` exists, modernizer offers to convert tickets into framework tickets at the end of analysis.

## Bundled MCP

The `auditor` agent uses the **context7** MCP server (declared in `tikkit/.mcp.json`) to verify libraries and frameworks against their *current* upstream docs — not the model's training cutoff. This catches things like deprecated CLI flags, removed APIs, and modern replacements (e.g., `setup.py` → `pyproject.toml + uv`).

## Technology Recommendations (2025)

See `references/languages/*.md` for detailed options with licensing and established/modern alternatives.

### Quick Reference (Production Defaults)

| Language | Package | Lint/Format | Test | CLI |
|----------|---------|-------------|------|-----|
| Python | uv | ruff | pytest | Typer |
| JS/TS Frontend | pnpm | Biome | Vitest | - |
| JS/TS Backend | pnpm + Node | Biome | Vitest | oclif |
| Java/Kotlin | Gradle 8+ | Spotless | JUnit 5 | picocli |
| Go | go mod | golangci-lint | go test | Cobra |
| Rust | cargo | clippy | built-in | clap |

> **Note:** Bun is recommended for CLI tools and scripts (fast startup), but Node.js remains the safer choice for production backend services due to better npm compatibility and stability.

### Language Reference Files

| Language | File | Key Tools |
|----------|------|-----------|
| Python | `python.md` | uv, ruff, ty, pytest, Typer |
| JS/TS Frontend | `javascript-frontend.md` | Vite, React, shadcn/ui, Vitest, Playwright |
| JS/TS Backend | `javascript-node.md` | Bun, Fastify, Elysia, oclif |
| Go | `go.md` | golangci-lint, Cobra, Gin, Viper |
| Rust | `rust.md` | clap, Axum, Tokio, criterion |
| Java/Kotlin | `java-kotlin.md` | Gradle, Spotless, Spring Boot, picocli |

## What This Skill Does NOT Do

- **Does not execute changes** — Only plans
- **Does not write code** — Delegates to agents
- **Does not assume tools exist** — Discovers what's available
- **Does not hardcode executors** — Matches tickets dynamically

## Support

**Author**: Zach Goldstein — Solutions Architect

**Issues**: [Report a bug](https://github.com/TheLampshady/tikkit/issues/new)
