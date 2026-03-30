---
name: improve-claude-md
description: Audit and optimize CLAUDE.md (or AGENTS.md) and CLAUDE.local.md files by removing discoverable content and focusing on preferences, behavioral nudges, and corrections. Reduces token cost and improves agent performance.
---

# Improve CLAUDE.md

Audit and optimize CLAUDE.md (or AGENTS.md) and CLAUDE.local.md files to maximize agent performance and minimize token waste.

## Principles

Based on research showing that including discoverable information (tech stack, key files, architecture, commands) in instruction files **hurts performance and increases cost by ~20%**. Agents can figure out project structure quickly on their own.

**What belongs in CLAUDE.md:**
- Preferences and behavioral nudges ("always do X", "never do Y")
- Corrections to default agent behavior
- Non-obvious conventions the agent can't infer from code
- User context (role, expertise level, communication preferences)
- Workflow preferences (tools to prefer, output formats)
- Conditional blocks for context-dependent behavior

**What does NOT belong:**
- Tech stack / framework listings (agent reads `package.json`, `Cargo.toml`, etc.)
- Key file locations / architecture maps (agent can explore)
- Script listings with flags (agent can run `--help`)
- MCP server descriptions (already pre-loaded via frontmatter)
- Skill descriptions (already pre-loaded via skill frontmatter)
- Tag/property/schema documentation (agent can read existing files to learn patterns)
- Folder structure descriptions (agent can `ls`)
- Standard command references (agent knows these)

## Invocation

```
/improve-claude-md
/improve-claude-md path/to/CLAUDE.md
```

Or natural language:
- "Audit my CLAUDE.md"
- "Optimize the agent instructions"
- "Slim down CLAUDE.md"

## Workflow

### Step 1: Read the target files

If no path is provided, look for `CLAUDE.md` (or `AGENTS.md`) in the project root. Also check for a `CLAUDE.local.md` in the same directory. If both exist, read both — they are audited together as a pair.

**Understanding the relationship:**
- `CLAUDE.md` — Checked into version control. Contains shared instructions applicable to all environments and contributors.
- `CLAUDE.local.md` — Git-ignored, private to the user's machine. Contains personal/local-only context (e.g., machine-specific paths, private workflow preferences, local-only conventions).

Both files are loaded into the agent's context at conversation start, so **the same optimization principles apply to both**. Redundancy between the two files also wastes tokens.

### Step 2: Categorize every section

For each section or block in both files (if both exist), classify it as one of:

| Category | Action | Examples |
|----------|--------|----------|
| **Preference** | KEEP | "Never delete a note", "Use Vikunja for personal tasks" |
| **Behavioral nudge** | KEEP | "Explain things simply", "Always test before sending URL" |
| **Non-obvious convention** | KEEP | "New notes go in vault root, not subfolders" |
| **User context** | KEEP | Role, expertise, goals, communication style |
| **Workflow override** | KEEP | Tool fallback order, specific tool preferences |
| **Conditional context** | KEEP (wrap in `<important if="...">`) | Different rules for different project areas |
| **Discoverable structure** | REMOVE | File trees, folder descriptions, "assets live in..." |
| **Tool/script reference** | REMOVE | Script names, flags, usage examples |
| **Tech stack listing** | REMOVE | "This project uses React, TypeScript..." |
| **Skill/MCP docs** | REMOVE | Descriptions of installed skills or MCP servers |
| **Schema/pattern docs** | REMOVE | Tag tables, frontmatter templates (unless non-obvious) |

### Step 2b: Check for cross-file issues

If both `CLAUDE.md` and `CLAUDE.local.md` exist, also check for:

- **Duplication** — Content repeated across both files wastes tokens. Flag it.
- **Misplaced content** — Shared conventions in `.local.md` (should be in `CLAUDE.md`) or private/machine-specific content in `CLAUDE.md` (should be in `.local.md`).
- **Contradictions** — Conflicting instructions between the two files.

**Placement guidance:**
- `CLAUDE.md` (shared/checked-in): Conventions, preferences, and instructions that apply regardless of environment or contributor.
- `CLAUDE.local.md` (private/git-ignored): Machine-specific paths, personal workflow preferences, local-only conventions, private context that shouldn't be shared.

### Step 3: Present the audit report

Show the user a report per file, plus cross-file findings if both exist:

```
## CLAUDE.md Audit Report

### Current size: X lines / ~Y tokens (CLAUDE.md) + X lines / ~Y tokens (CLAUDE.local.md)

### Sections to KEEP (preferences & nudges):
- [File] [Section name] — [why it's valuable]

### Sections to REMOVE (discoverable):
- [File] [Section name] — [how agent discovers this instead]

### Sections to CONDENSE:
- [File] [Section name] — [what to keep, what to cut]

### Cross-file issues:
- [Duplicated content between files]
- [Content in wrong file — suggest moving]
- [Contradictions]

### Suggestions:
- [Conditional blocks that would help]
- [Missing nudges that would be useful]

### Estimated reduction: ~X% fewer tokens
```

### Step 4: Wait for user approval

Do NOT modify the file without explicit approval. Present the report and ask:
- "Apply all suggestions?"
- "Apply selectively?" (let user pick)
- "Show me the proposed version first?"

### Step 5: Apply changes

When approved:
1. Create backups: copy originals to `CLAUDE.md.YYYY-MM-DD.backup` (and `CLAUDE.local.md.YYYY-MM-DD.backup` if it was audited)
2. Rewrite the file(s) with only the kept/condensed content
3. Move any misplaced content between files as approved
4. Show a before/after summary (line count per file, estimated token reduction)

## Guidelines for edge cases

- **Frontmatter examples**: Remove if agent can learn from existing files. Keep if the convention is non-obvious or counter-intuitive.
- **Integration conventions** (e.g., bidirectional linking between tools): Keep — these encode decisions the agent can't infer.
- **Proactive workflows** (scheduled reports, digest formats): Keep — these are behavioral instructions.
- **"About the user" sections**: Keep and potentially expand — this is high-value context.
- **Conditional blocks**: Suggest wrapping context-dependent rules in `<important if="context">` blocks.
- **Links to external docs**: Keep if they point to resources the agent should consult. Remove if they just document what's already installed.
