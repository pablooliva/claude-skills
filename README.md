# SDD Flow

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that orchestrates end-to-end feature development using **Specification-Driven Development (SDD)**.

SDD Flow takes a task or software requirement and drives it through a structured lifecycle — **Research, Planning, Implementation** — with adversarial reviews at every stage. Each phase runs as a separate subagent with a fresh context window, and all coordination happens through artifact files on disk.

This skill is the orchestration layer for the [SDD plugin](https://github.com/pablooliva/claude-plugins). The plugin provides the individual phase commands (`/sdd:research-start`, `/sdd:planning-start`, `/sdd:implementation-start`, etc.), while SDD Flow chains them together into a fully automated, end-to-end workflow with review gates between each phase.

## Why SDD Flow?

AI coding assistants are powerful, but they often jump straight to implementation. SDD Flow enforces discipline:

- **Research first** — Investigate the codebase, constraints, and edge cases before writing a line of code.
- **Spec as source of truth** — A formal specification bridges research findings and implementation decisions, so nothing gets lost.
- **Mandatory reviews** — Every phase goes through adversarial review. All findings (HIGH, MEDIUM, LOW) must be resolved before moving forward. Reviews are gates, not checkboxes.
- **Full traceability** — Every implementation decision traces back to a spec requirement, which traces back to a research finding.

## How It Works

```
/sdd-flow Add GDPR-compliant audit logging for all anonymization requests
```

SDD Flow spawns a sequence of subagents, each handling one step:

### 1. Research Phase
- A research subagent investigates the codebase, dependencies, and constraints
- A completeness subagent validates the research and fills gaps
- A critical review subagent performs an adversarial review
- A fix subagent resolves all review findings
- Artifacts are committed

### 2. Planning Phase
- A planning subagent reads the research and writes a formal specification (SPEC)
- A completeness subagent validates the spec
- A critical review subagent checks for ambiguities, untestable criteria, and dropped findings
- A fix subagent resolves all review findings
- Artifacts are committed

### 3. Implementation Phase
- An implementation subagent builds all code and tests per the spec
- A code review subagent performs spec-driven review (70% spec alignment, 20% context engineering, 10% test alignment)
- A fix subagent addresses code review findings
- A critical review subagent performs adversarial implementation review
- A second fix pass resolves remaining findings
- A completion subagent finalizes documentation and creates an implementation summary
- All code and artifacts are committed

## Execution Modes

| Mode | Flag | Behavior |
|------|------|----------|
| **Supervised** (default) | `--supervised` | Pauses for your approval after research and before final commit |
| **Autonomous** | `--auto` | Runs the full lifecycle without stopping |

## Usage

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

## Artifacts

SDD Flow produces structured artifacts in an `SDD/` directory:

```
SDD/
├── research/          # Research documents
├── requirements/      # Specifications (SPEC files)
├── prompts/           # Implementation tracking and summaries
│   ├── implementation-complete/
│   └── context-management/
│       └── progress.md
└── reviews/           # Critical reviews and code reviews
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- The [SDD plugin](https://github.com/pablooliva/claude-plugins) installed (`claude plugin add pablooliva/sdd`)

## Installation

Copy the `sdd-flow/` directory into your Claude Code skills directory:

```bash
cp -r sdd-flow/ ~/.claude/skills/sdd-flow/
```

Or clone this repo and symlink it:

```bash
git clone <this-repo-url>
ln -s "$(pwd)/sdd-flow" ~/.claude/skills/sdd-flow
```

## Key Principles

1. Each phase must be thorough — don't rush research to get to implementation
2. Research informs planning, planning constrains implementation
3. Every requirement gets a test — no exceptions
4. Reviews are mandatory and all findings must be resolved before proceeding
5. The spec is the source of truth for implementation

## License

[MIT](LICENSE)
