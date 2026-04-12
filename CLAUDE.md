# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.

## What This Repo Is

A library of Claude Code **agent definitions** (`agents/`) and **skills** (`skills/`) — reusable `.md` files that can be dropped into any project. Also contains Go-specific coding rules (`rules/golang/`) that agents reference at review time.

## File Conventions

### Agents (`agents/<name>.md`)

Frontmatter fields:

```yaml
---
name: agent-name
description: >
  One-paragraph description used by Claude to decide when to invoke this agent.
  Trigger conditions go here — be explicit (e.g., "Use PROACTIVELY when…").
tools: ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
model: sonnet   # or opus / haiku — match capability to task
---
```

- `description` is the primary trigger signal; write it as if explaining to another Claude when to use this agent.
- Only list tools the agent actually needs.
- Use `opus` only for tasks that genuinely need deep reasoning (planning, architecture). Default to `sonnet`.

### Skills (`skills/<name>/SKILL.md`)

Frontmatter fields:

```yaml
---
name: skill-name
description: One-line summary of what the skill provides
origin: ECC    # optional — source attribution
---
```

Skills are invoked as slash commands (`/skill-name`). They should explain *when to activate* at the top so Claude knows to load them proactively.

### Rules (`rules/golang/*.md`)

Plain markdown — no frontmatter. Agents load these explicitly at review time (see `agents/code-reviewer.md` for the pattern). Rules files are the authoritative standard; they override any default Claude behavior.

## Architecture

The `code-reviewer` agent demonstrates the intended pattern for rules-driven agents:

1. **Load rules first** — reads `rules/golang/{coding-style,security,performance,testing}.md` before touching any diff.
2. **Structured output** — every finding references the specific rule file and section that was violated.
3. **Severity tiers** — CRITICAL / HIGH / MEDIUM / LOW with fixed approval criteria (block on CRITICAL, warn on HIGH).

Other agents follow a similar pattern: read context files first, produce structured output, stay within defined scope.

## Adding New Content

- **New agent**: copy an existing agent as a template; write the `description` trigger carefully.
- **New skill**: create `skills/<name>/SKILL.md`; include a "When to Activate" section.
- **New Go rule**: add to the appropriate file in `rules/golang/`; the `code-reviewer` agent will pick it up automatically.
- Prefer editing an existing rule file over creating a new one unless the topic is clearly distinct.
