---
name: tik
description: "Create structured tickets from requests. Use this skill when the user asks to: create a ticket, write a ticket, make a ticket, draft a task, write up a feature request, create a story, write a bug report, or turn a request into a ticket. Also triggers when the user says things like 'ticket this', 'can you write that up', 'make a card for this', 'I need a ticket for X'. This is the default ticket creation skill — use it for any ticket request that does NOT involve a Figma link or Stitch exports. If the user provides a Figma URL or mentions Figma, use figtik instead. If the user references Stitch exports or a stitch/ directory, use stitchtik instead."
user-invocable: true
---

# tik

Turn simple requests into structured, well-written tickets. The output should be useful to everyone on the team — product leads understand the Overview, engineers work from Goals and Tech Details, and QA validates against Acceptance Criteria.

## When to Use

This is the default ticket creation skill. The user might say anything from "I need a ticket for adding dark mode" to a detailed paragraph about a bug they found. Your job is to expand their request into a complete ticket.

If the user provides a Figma link or wants a ticket based on a Figma design, hand off to `/figtik`. If they reference Stitch exports or a `stitch/` directory, hand off to `/stitchtik`. If they want a codebase audit, redirect to `/modernizer`.

## Process

### 1. Understand the Request

Read the user's request carefully. Identify:
- **What** they want built, fixed, or changed
- **Why** it matters (business context, user impact, urgency)
- **Any technical constraints** they mentioned (specific services, APIs, frameworks)
- **Any links or files** they referenced

### 2. Research Before Asking

Before asking the user anything, check what's already there:

- **Check `.backlog/backlog.md` for duplicate intent.** The match isn't just slug — it's the user's actual goal. Two tickets phrased differently can still be duplicates. If you find a match, surface it instead of creating a new ticket.
- **Read relevant source files, configs, READMEs, and existing docs** to understand what currently exists, what's already wired up, and what conventions are in place.
- **Skim related tickets in `.backlog/tickets/`** to align tone, slug style, and avoid contradicting prior decisions.

This reduces back-and-forth and grounds the ticket in the actual state of the project.

### 3. Clarify Only What You Can't Resolve

If questions remain after researching, ask the user — but do it well:

- **Offer a recommendation with each question.** Don't just ask "which database?" — say "I'd recommend PostgreSQL here because the project already uses it for X, but Redis could work if read latency is the priority. Which direction?"
- **Batch questions** — 2-3 max per round.
- **Leave some things open.** Questions like "which caching strategy?" or "should we use a new service or extend the existing one?" don't have to block ticket creation. Flag them under Tech Details → Open Questions so a downstream SDD framework (e.g., speckit) can resolve them during spec generation.

### 4. Compose the Ticket

Read the canonical template at `./references/ticket-template.md`. Use it as-is — tik uses the base format with no additional sub-sections.

Every ticket gets Overview, Goals, and Acceptance Criteria. Tech Details, References, and Other appear only when there's something real to put there — don't pad with empty sections. Use `[TBD]` placeholders for unknowns; the downstream SDD framework will resolve these during spec generation.

### 5. Save and Report

Save the ticket to `.backlog/tickets/<slug>.md`:

- **Generate a kebab-case slug** from the title — e.g., `inventory-search-filter`, `dark-mode-toggle`. No prefixes, no dates.
- **Resolve collisions with intent, not numbers.** If `.backlog/tickets/<slug>.md` already exists, extend the slug with a qualifier that captures what makes this ticket different — `dark-mode-toggle-mobile` vs. `dark-mode-toggle-desktop`, not `dark-mode-toggle-2`.
- **Create `.backlog/tickets/` if it doesn't exist.**
- **Append a line to `.backlog/backlog.md`:** `- [ ] <Title> [tik] → tickets/<slug>.md`. Create `.backlog/backlog.md` if it doesn't exist — it's the master index that other tikkit skills (and downstream tools) read.

Then give the user a brief summary in the conversation:
- Title and file path
- 1-2 sentences on what the ticket covers
- Any `[TBD]` items or open questions flagged for the spec process

---

## Example

**User:** "I need a ticket for adding a dark mode toggle to the settings page."

**After researching:** `src/styles/theme.ts` already defines light/dark variants but exposes no UI; `src/components/Toggle.tsx` is the standard toggle component; no existing tickets in `.backlog/backlog.md` cover dark mode.

**Output — `.backlog/tickets/dark-mode-toggle.md`:**

```markdown
# Dark Mode Toggle

## Overview

Add a user-facing toggle in the settings page that switches the app between light and dark themes. The dark theme already exists in code but is not currently exposed to users.

## Goals

* **UI**
  - Add a "Theme" row to the settings page under Appearance
  - Use the existing `Toggle` component (`src/components/Toggle.tsx`)
  - Persist the user's choice across sessions

* **Behavior**
  - Wire the toggle to the existing theme provider in `src/styles/theme.ts`
  - Default to system preference (`prefers-color-scheme`) on first visit

## Tech Details

Light/dark variants already exist in `src/styles/theme.ts`. This is a UI exposure ticket, not a theming implementation.

**Open Questions (for spec process):**
- Persistence layer: localStorage vs. user profile API? [TBD]

## Acceptance Criteria

* **Given:** the user is on the settings page
  **When:** they toggle "Dark Mode" on
  **Then:** the entire app switches to the dark theme
  **and Then:** the preference persists after page reload

* **Given:** a user with no saved preference
  **When:** they first load the app
  **Then:** the theme matches their OS-level `prefers-color-scheme`
```

**Reply in chat:** "Created `.backlog/tickets/dark-mode-toggle.md` and added it to `.backlog/backlog.md`. Covers the toggle UI, persistence, and OS-preference fallback. One open question flagged: localStorage vs. user profile for persistence."

---

## Writing Guidelines

The bundled template (`./references/ticket-template.md`) carries section-by-section guidance. A few things that matter most for tik:

- **Make Goals concrete.** "Add search functionality" isn't a goal. "Add a search bar to the inventory page that filters by name, SKU, and category as the user types" is. If the request was vague, make defensible assumptions and state them in the ticket — explicit assumptions are easier to challenge than implicit ones.

- **Acceptance Criteria belong to QA.** Each scenario must be independently testable. Use Given/When/Then:
  - **Given** — the starting state
  - **When** — a single action
  - **Then** — the expected outcome
  - **and Then** — additional outcomes from the same action; chain instead of duplicating the Given/When

  Lead with the happy path, then edge cases and error states.

- **Don't manufacture references.** Only include links and file paths that actually exist. An empty References section beats one full of plausible-sounding URLs that don't resolve.

- **Use `[TBD]` instead of stalling.** Open technical questions can ride along under Tech Details → Open Questions. Flag them clearly so the downstream SDD framework picks them up rather than re-discovering them.

## Tone

Write tickets in a professional but approachable tone. Be direct. Don't pad with filler. The ticket should feel like it was written by someone who understands the work and respects the reader's time.
