# Claude Skills

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for structured, repeatable workflows.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [SDD Flow](#sdd-flow) | `/sdd-flow` | End-to-end feature development using Specification-Driven Development |
| [Improve CLAUDE.md](#improve-claudemd) | `/improve-claude-md` | Audit and optimize CLAUDE.md files to reduce token waste |

## Installation

Copy individual skill directories into your Claude Code skills directory:

```bash
cp -r <skill-name>/ ~/.claude/skills/<skill-name>/
```

Or clone the repo and symlink:

```bash
git clone <this-repo-url>
ln -s "$(pwd)/<skill-name>" ~/.claude/skills/<skill-name>
```

---

## SDD Flow

Orchestrates end-to-end feature development using **Specification-Driven Development (SDD)**. Takes a task or software requirement and drives it through a structured lifecycle — **Research, Planning, Implementation** — with adversarial reviews at every stage. Each phase runs as a separate subagent with a fresh context window, and all coordination happens through artifact files on disk.

This skill is the orchestration layer for the [SDD plugin](https://github.com/pablooliva/claude-plugins). The plugin provides the individual phase commands (`/sdd:research-start`, `/sdd:planning-start`, `/sdd:implementation-start`, etc.), while SDD Flow chains them together into a fully automated, end-to-end workflow with review gates between each phase.

### Why SDD Flow?

AI coding assistants are powerful, but they often jump straight to implementation. SDD Flow enforces discipline:

- **Research first** — Investigate the codebase, constraints, and edge cases before writing a line of code.
- **Spec as source of truth** — A formal specification bridges research findings and implementation decisions, so nothing gets lost.
- **Mandatory reviews** — Every phase goes through adversarial review. All findings (HIGH, MEDIUM, LOW) must be resolved before moving forward.
- **Full traceability** — Every implementation decision traces back to a spec requirement, which traces back to a research finding.

### How It Works

```
/sdd-flow Add GDPR-compliant audit logging for all anonymization requests
```

SDD Flow spawns a sequence of subagents, each handling one step:

**0. Scope Assessment**
- Analyzes whether the feature request fits in a single SDD cycle
- If too large, produces a decomposition checklist of smaller, independently deliverable features
- The user then runs `/sdd-flow` for each checklist item at their own pace
- Small requests pass through this gate transparently

**1. Research Phase**
- A research subagent investigates the codebase, dependencies, and constraints
- A completeness subagent validates the research and fills gaps
- A critical review subagent performs an adversarial review
- A fix subagent resolves all review findings
- Artifacts are committed

**2. Planning Phase**
- A planning subagent reads the research and writes a formal specification (SPEC)
- A completeness subagent validates the spec
- A critical review subagent checks for ambiguities, untestable criteria, and dropped findings
- A fix subagent resolves all review findings
- Artifacts are committed

**3. Implementation Phase**
- An implementation subagent builds all code and tests per the spec
- A code review subagent performs spec-driven review (70% spec alignment, 20% context engineering, 10% test alignment)
- A fix subagent addresses code review findings
- A critical review subagent performs adversarial implementation review
- A second fix pass resolves remaining findings
- A completion subagent finalizes documentation and creates an implementation summary
- All code and artifacts are committed

### Execution Modes

| Mode | Flag | Behavior |
|------|------|----------|
| **Supervised** (default) | `--supervised` | Pauses for your approval after research and before final commit |
| **Autonomous** | `--auto` | Runs the full lifecycle without stopping |

### Usage

```bash
# Basic usage
/sdd-flow <task description>

# With a ticket/issue number
/sdd-flow #42 Add CSV export to the reports page

# Autonomous mode
/sdd-flow --auto Implement allow-list management UI

# Resume interrupted work
/sdd-flow continue
```

### Artifacts

SDD Flow produces structured artifacts in an `SDD/` directory:

```
SDD/
├── flow/              # Scope decomposition documents
├── research/          # Research documents
├── requirements/      # Specifications (SPEC files)
├── prompts/           # Implementation tracking and summaries
│   ├── implementation-complete/
│   └── context-management/
│       └── progress.md
└── reviews/           # Critical reviews and code reviews
```

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- The [SDD plugin](https://github.com/pablooliva/claude-plugins) installed (`claude plugin add pablooliva/sdd`)

---

## Improve CLAUDE.md

Audits and optimizes `CLAUDE.md` (or `AGENTS.md`) and `CLAUDE.local.md` files to maximize agent performance and minimize token waste.

Based on research showing that including discoverable information (tech stack, key files, architecture, commands) in instruction files **hurts performance and increases cost by ~20%**. Agents can figure out project structure on their own — your instruction files should focus on what they *can't* infer.

### What Belongs in CLAUDE.md

| Keep | Remove |
|------|--------|
| Preferences and behavioral nudges | Tech stack / framework listings |
| Corrections to default agent behavior | Key file locations / architecture maps |
| Non-obvious conventions | Script listings with flags |
| User context (role, expertise, communication style) | MCP server / skill descriptions |
| Workflow preferences (tool choices, output formats) | Schema / pattern documentation |
| Conditional behavior blocks | Folder structure descriptions |

### Usage

```bash
# Audit CLAUDE.md in the current project
/improve-claude-md

# Audit a specific file
/improve-claude-md path/to/CLAUDE.md
```

### Workflow

1. **Read** — Finds and reads `CLAUDE.md` and `CLAUDE.local.md` (if both exist)
2. **Categorize** — Classifies every section as keep, remove, or condense
3. **Cross-file check** — Flags duplication, misplaced content, and contradictions between the two files
4. **Report** — Presents an audit report with estimated token reduction
5. **Apply** — After your approval, creates backups and rewrites the file(s)

Changes are never applied without explicit approval.

---

## License

[MIT](LICENSE)
