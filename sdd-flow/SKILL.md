> **⚠️ ARCHIVED — This skill has moved.**
> This standalone skill is no longer maintained. It is now part of the [agent-engineering plugin](https://github.com/pablooliva/claude-plugins/tree/main/agent-engineering) — please install the plugin to get the latest version and ongoing updates.

# SDD Flow — End-to-End Feature Development

Takes a task or software requirement and drives it through the complete SDD (Specification-Driven Development) lifecycle: **Research → Planning → Implementation → Done**.

All phases run on **Claude Opus** by default. No model switching required. Phases are executed via subagents, giving each phase a fresh context window.

## Usage

```
/sdd-flow <task or requirement description>
```

You can also provide a ticket/issue number:

```
/sdd-flow #42 Add CSV export to the reports page
```

## How This Skill Works

This skill uses **subagents** to execute each phase of the SDD workflow. The main conversation acts as a lightweight orchestrator — it spawns subagents for research, planning, implementation, reviews, and fixes. Each subagent gets a fresh context window, eliminating the need for manual session clears.

All inter-phase communication happens through the **SDD artifact files on disk** (see Artifact Paths Contract below). Every subagent is given explicit paths for reading inputs and writing outputs.

**The SDD plugin must be installed.** Subagents receive the equivalent instructions from the plugin's commands embedded directly in their prompts (since subagents cannot invoke slash commands).

**SDD plugin location:** `~/.claude/plugins/cache/pablooliva/sdd/` — read command files from the `commands/` subdirectory within the latest version (e.g., `~/.claude/plugins/cache/pablooliva/sdd/1.0.0/commands/`). Do NOT confuse this with other plugins (e.g., PACE) that have similarly named commands.

## CRITICAL: Model Override

The SDD plugin commands contain model checks that require Opus for research and Sonnet for planning/implementation. **This skill overrides that behavior.**

When embedding SDD command instructions into subagent prompts, **strip all model verification steps.** Specifically:
- Remove any "This command requires Claude Sonnet/Opus" checks
- Remove any "STOP all further processing" instructions related to model verification
- Remove any warnings about switching models

All three phases run on whatever model is currently active — **Opus is the intended default for all phases**.

---

## Artifact Paths Contract

Every subagent MUST use these exact paths. The orchestrator MUST include the resolved paths (with actual values for `[###]`, `[feature-name]`, dates, etc.) in every subagent prompt.

### Canonical Identifiers (resolved at Step 0)

| Identifier | Description | Example |
|------------|-------------|---------|
| `[###]` | Issue/ticket number or sequential ID | `042` |
| `[feature-name]` | Kebab-case feature name | `audit-logging` |
| `[YYYY-MM-DD]` | Current date | `2026-03-29` |
| `[YYYY-MM-DD_HH-MM-SS]` | Timestamp (24h, underscores) | `2026-03-29_14-30-45` |

### Phase Artifacts — Exact Paths

| Artifact | Path | Created By | Read By |
|----------|------|------------|---------|
| **Scope decomposition** | `SDD/flow/DECOMPOSITION-[###]-[feature-name].md` | Scope assessment subagent | User (manual `/sdd-flow` per item) |
| **Research document** | `SDD/research/RESEARCH-[###]-[feature-name].md` | Research subagent | Research review, Planning subagent |
| **Research critical review** | `SDD/reviews/CRITICAL-RESEARCH-[feature-name]-[YYYYMMDD].md` | Research review subagent | Research fix subagent |
| **Specification** | `SDD/requirements/SPEC-[###]-[feature-name].md` | Planning subagent | Planning review, Implementation subagent |
| **Spec critical review** | `SDD/reviews/CRITICAL-SPEC-[feature-name]-[YYYYMMDD].md` | Planning review subagent | Planning fix subagent |
| **PROMPT tracking doc** | `SDD/prompts/PROMPT-[###]-[feature-name]-[YYYY-MM-DD].md` | Implementation subagent | Code review, Impl review, Completion subagent |
| **Code review** | `SDD/reviews/REVIEW-[###]-[feature-name]-[YYYYMMDD].md` | Code review subagent | Implementation fix subagent |
| **Impl critical review** | `SDD/reviews/CRITICAL-IMPL-[feature-name]-[YYYYMMDD].md` | Impl review subagent | Implementation fix subagent |
| **Implementation summary** | `SDD/prompts/implementation-complete/IMPLEMENTATION-SUMMARY-[###]-[YYYY-MM-DD_HH-MM-SS].md` | Completion subagent | — |
| **Progress file** | `SDD/prompts/context-management/progress.md` | All subagents (append only) | All subagents |

### Directory Structure

```
SDD/
├── flow/
│   └── DECOMPOSITION-[###]-[feature-name].md
├── research/
│   └── RESEARCH-[###]-[feature-name].md
├── requirements/
│   └── SPEC-[###]-[feature-name].md
├── prompts/
│   ├── PROMPT-[###]-[feature-name]-[YYYY-MM-DD].md
│   ├── implementation-complete/
│   │   └── IMPLEMENTATION-SUMMARY-[###]-[YYYY-MM-DD_HH-MM-SS].md
│   └── context-management/
│       ├── progress.md
│       └── subagent-calls/
└── reviews/
    ├── CRITICAL-RESEARCH-[feature-name]-[YYYYMMDD].md
    ├── CRITICAL-SPEC-[feature-name]-[YYYYMMDD].md
    ├── CRITICAL-IMPL-[feature-name]-[YYYYMMDD].md
    └── REVIEW-[###]-[feature-name]-[YYYYMMDD].md
```

### Subagent Path Rules

Every subagent prompt MUST include:
1. The **resolved** paths for all artifacts it needs to read (inputs)
2. The **resolved** paths for all artifacts it must write (outputs)
3. An instruction to **verify input files exist** before starting work (read them; if missing, fail with a clear error message stating which file is missing and what path was expected)
4. An instruction to **create parent directories** before writing output files (`mkdir -p` as needed)
5. An instruction to **append** to `progress.md`, never overwrite or delete existing content

---

## Execution Modes

### Step 0: Scope Assessment

When invoked, first:

1. Extract the **task description** from the user's input
2. Extract **issue/ticket number** if provided (e.g., `#42`, `PROJ-123`), otherwise determine sequential numbering by checking existing SDD artifacts
3. Derive a **kebab-case feature name** from the task
4. Resolve all canonical identifiers (see Artifact Paths Contract)

Then spawn a **general-purpose subagent** for scope assessment:

- **Inputs:** Task description, codebase access
- **Outputs:** Either a decomposition document or a "proceed" signal
- **Task:** Analyze the requested feature and determine whether it can be completed in a single SDD cycle (research → planning → implementation) or whether it should be decomposed into smaller, independently deliverable chunks.

#### Assessment Heuristics

The subagent should consider:

- **Number of distinct components or systems touched** — A feature that modifies one module is different from one that cuts across the API layer, database schema, frontend, and background jobs.
- **Number of independent user-facing behaviors** — Multiple distinct behaviors (e.g., "add CSV export AND add scheduled reports AND add a dashboard widget") are likely separate features.
- **Natural seams** — Can the feature be split at boundaries where each piece delivers standalone value? If so, it probably should be.
- **Specification complexity** — Would the resulting SPEC document have so many requirements that a single implementation subagent couldn't hold them all in context?
- **Test surface** — Would the test suite for this feature require testing multiple unrelated subsystems?

This is a judgment call, not a formula. The subagent should explain its reasoning.

#### If the scope is manageable (single SDD cycle)

The subagent reports that the feature fits in one cycle. The orchestrator proceeds directly to **Step 1: Parse Input and Select Mode** with no pause — the user should not notice this gate for small requests.

#### If the scope is too large (decomposition needed)

The subagent produces a decomposition document at:

```
SDD/flow/DECOMPOSITION-[###]-[feature-name].md
```

This document contains:

1. **Rationale** — Why this feature request is too large for a single SDD cycle, referencing the heuristics above.
2. **Decomposition checklist** — An ordered list of smaller, independently deliverable features. Each item includes:
   - A clear, self-contained task description (suitable as input to `/sdd-flow`)
   - A brief note on what it delivers and why it's sequenced where it is
   - Any dependencies on prior checklist items
3. **Dependency map** — Which items must be completed before others (some may be parallelizable).

The orchestrator presents the decomposition to the user:

> **This feature request is too large for a single SDD cycle.**
>
> I've broken it into [N] independently deliverable steps:
>
> [Checklist summary]
>
> Full decomposition: `SDD/flow/DECOMPOSITION-[###]-[feature-name].md`
>
> Review and edit the decomposition as needed, then run `/sdd-flow` for each item when ready. Items marked with dependencies should be completed in order.

The skill then **stops**. The user manually invokes `/sdd-flow <checklist item description>` for each item at their own pace.

---

### Step 1: Parse Input and Select Mode

**Note:** This step is reached only after Step 0 determines the feature fits in a single SDD cycle.

**Ask the user which execution mode they want:**

> **Choose execution mode:**
>
> **Supervised** (default) — I'll run autonomously but pause for your approval at two checkpoints:
> 1. After research is complete (so you can confirm direction before planning/implementation)
> 2. Before committing implementation (so you can review the code)
>
> **Autonomous** — Fully autonomous, no checkpoints. I'll run research → planning → implementation → done without stopping. You'll see the final result when everything is complete.
>
> Reply **s** for supervised or **a** for autonomous. (Default: supervised)

If the user's original invocation already includes a mode flag (e.g., `/sdd-flow --auto <task>` or `/sdd-flow --supervised <task>`), skip the prompt and use that mode.

---

## Orchestration Instructions

The orchestrator (main conversation) spawns subagents sequentially. Each subagent receives:
- The full instructions from the corresponding SDD plugin command (with model checks stripped)
- Resolved artifact paths for its inputs and outputs
- The task description and canonical identifiers
- Any relevant context from previous subagent results

### Step 2: Research Phase

#### 2a. Research Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:research-start` command (embedded in prompt, model checks stripped)
- **Inputs:** Task description, codebase access
- **Outputs:** `SDD/research/RESEARCH-[###]-[feature-name].md`, update `progress.md`
- **Task:** Create the research document and perform the full systematic investigation

Then spawn a second **general-purpose subagent** with:
- **Instructions from:** `/sdd:research-complete` command
- **Inputs:** The RESEARCH document at its exact path
- **Outputs:** Updated RESEARCH document (if gaps found), updated `progress.md`
- **Task:** Validate completeness against the checklist, fill any remaining gaps

#### 2b. Research Critical Review Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:critical-review` command (research phase section)
- **Inputs:** `SDD/research/RESEARCH-[###]-[feature-name].md`
- **Outputs:** `SDD/reviews/CRITICAL-RESEARCH-[feature-name]-[YYYYMMDD].md`
- **Task:** Adversarial review of the research document

#### 2c. Address Research Review Findings

Spawn a **general-purpose subagent** with:
- **Inputs:** `SDD/research/RESEARCH-[###]-[feature-name].md` AND `SDD/reviews/CRITICAL-RESEARCH-[feature-name]-[YYYYMMDD].md`
- **Outputs:** Updated `SDD/research/RESEARCH-[###]-[feature-name].md`, updated `progress.md`
- **Task:** Resolve ALL findings from the critical review — HIGH, MEDIUM, and LOW severity. Update the RESEARCH document to fill gaps, strengthen weak evidence, add missing perspectives, and address questionable assumptions. No finding is left unresolved. After fixing, append a "Findings Addressed" section to the review document noting how each finding was resolved.

#### 2d. Commit Research Artifacts

The **orchestrator** runs the commit (not a subagent), following `/sdd:commit` conventions — no co-author attribution.

#### 2e. Supervised Checkpoint (if supervised mode)

If in **supervised mode**, pause and present a summary to the user:

> **Research phase complete.** Here's what was found:
> [Brief summary of key research findings and critical review results]
>
> Research document: `SDD/research/RESEARCH-[###]-[feature-name].md`
> Critical review: `SDD/reviews/CRITICAL-RESEARCH-[feature-name]-[YYYYMMDD].md`
>
> **Proceed to planning?** (y/n)

Wait for user confirmation before proceeding to Step 3.

If in **autonomous mode**, proceed directly to Step 3.

---

### Step 3: Planning Phase

#### 3a. Planning Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:planning-start` command (model checks stripped)
- **Inputs:** `SDD/research/RESEARCH-[###]-[feature-name].md`, `SDD/prompts/context-management/progress.md`
- **Outputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, updated `progress.md`
- **Task:** Read the research document and create the full specification

Then spawn a second **general-purpose subagent** with:
- **Instructions from:** `/sdd:planning-complete` command
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, `SDD/research/RESEARCH-[###]-[feature-name].md`
- **Outputs:** Updated SPEC document (if gaps found), updated `progress.md`
- **Task:** Validate completeness against the checklist, ensure all research findings are incorporated

#### 3b. Specification Critical Review Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:critical-review` command (planning phase section)
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, `SDD/research/RESEARCH-[###]-[feature-name].md`
- **Outputs:** `SDD/reviews/CRITICAL-SPEC-[feature-name]-[YYYYMMDD].md`
- **Task:** Adversarial review of the specification, checking for ambiguities, untestable criteria, dropped research findings, contradictions

#### 3c. Address Specification Review Findings

Spawn a **general-purpose subagent** with:
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md` AND `SDD/reviews/CRITICAL-SPEC-[feature-name]-[YYYYMMDD].md` AND `SDD/research/RESEARCH-[###]-[feature-name].md`
- **Outputs:** Updated `SDD/requirements/SPEC-[###]-[feature-name].md`, updated `progress.md`
- **Task:** Resolve ALL findings — clarify ambiguous requirements, make criteria testable, add missing edge cases, resolve contradictions, incorporate dropped research findings. Append "Findings Addressed" section to the review document.

#### 3d. Commit Planning Artifacts

The **orchestrator** runs the commit.

#### 3e. Transition

Proceed directly to Step 4 (no checkpoint needed here — the supervised checkpoint covers the most critical decision point at research, and the second checkpoint comes before final implementation commit).

---

### Step 4: Implementation Phase

#### 4a. Implementation Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:implementation-start` command (model checks stripped)
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, `SDD/research/RESEARCH-[###]-[feature-name].md`, `SDD/prompts/context-management/progress.md`
- **Outputs:** `SDD/prompts/PROMPT-[###]-[feature-name]-[YYYY-MM-DD].md`, implemented code and tests, updated `progress.md`
- **Task:** Read the specification and implement ALL requirements:
  - Core functionality (happy path)
  - Edge cases (EDGE-XXX from spec)
  - Failure handling (FAIL-XXX from spec)
  - Tests alongside each component
  - Performance and security validation
  - Update PROMPT tracking document throughout

**Note:** If the implementation is too large for a single subagent's context, the orchestrator should split it into multiple sequential subagents — each handling a subset of requirements. Each subsequent subagent reads the updated PROMPT document to understand what's been completed.

#### 4b. Code Review Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:code-review` command
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, `SDD/research/RESEARCH-[###]-[feature-name].md`, `SDD/prompts/PROMPT-[###]-[feature-name]-[YYYY-MM-DD].md`, the implemented code files (paths from PROMPT document)
- **Outputs:** `SDD/reviews/REVIEW-[###]-[feature-name]-[YYYYMMDD].md`
- **Task:** Specification-driven code review (70% spec alignment, 20% context engineering, 10% test alignment)

#### 4c. Address Code Review Findings

Spawn a **general-purpose subagent** with:
- **Inputs:** `SDD/reviews/REVIEW-[###]-[feature-name]-[YYYYMMDD].md`, `SDD/requirements/SPEC-[###]-[feature-name].md`, the implemented code files
- **Outputs:** Updated code and tests, updated PROMPT document, "Findings Addressed" appended to review document
- **Task:** Fix ALL findings until the implementation meets APPROVED status. Resolve specification misalignment, missing edge/failure handling, test gaps, and all other issues.

#### 4d. Implementation Critical Review Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:critical-review` command (implementation phase section)
- **Inputs:** `SDD/requirements/SPEC-[###]-[feature-name].md`, implemented code files, test files
- **Outputs:** `SDD/reviews/CRITICAL-IMPL-[feature-name]-[YYYYMMDD].md`
- **Task:** Adversarial review of the implementation

#### 4e. Address Implementation Review Findings

Spawn a **general-purpose subagent** with:
- **Inputs:** `SDD/reviews/CRITICAL-IMPL-[feature-name]-[YYYYMMDD].md`, `SDD/requirements/SPEC-[###]-[feature-name].md`, implemented code files
- **Outputs:** Updated code and tests, updated PROMPT document, "Findings Addressed" appended to review document
- **Task:** Resolve ALL findings — fix specification deviations, security vulnerabilities, silent failures, missing test coverage, and every other issue regardless of severity.

#### 4f. Implementation Completion Subagent

Spawn a **general-purpose subagent** with:
- **Instructions from:** `/sdd:implementation-complete` command (model checks stripped)
- **Inputs:** `SDD/prompts/PROMPT-[###]-[feature-name]-[YYYY-MM-DD].md`, `SDD/requirements/SPEC-[###]-[feature-name].md`
- **Outputs:** Updated PROMPT document, updated SPEC document, `SDD/prompts/implementation-complete/IMPLEMENTATION-SUMMARY-[###]-[YYYY-MM-DD_HH-MM-SS].md`, updated `progress.md`
- **Task:** Finalize all documentation, validate all requirements are met, create implementation summary

#### 4g. Supervised Checkpoint (if supervised mode)

If in **supervised mode**, pause and present a summary to the user:

> **Implementation complete.** Here's a summary:
> [Brief summary of what was built, test results, review outcomes]
>
> Key artifacts:
> - Spec: `SDD/requirements/SPEC-[###]-[feature-name].md`
> - Code review: `SDD/reviews/REVIEW-[###]-[feature-name]-[YYYYMMDD].md`
> - Critical review: `SDD/reviews/CRITICAL-IMPL-[feature-name]-[YYYYMMDD].md`
> - Implementation summary: `SDD/prompts/implementation-complete/IMPLEMENTATION-SUMMARY-[###]-[timestamp].md`
>
> **Ready to commit all implementation code?** (y/n)

Wait for user confirmation before committing.

If in **autonomous mode**, proceed directly to commit.

#### 4h. Commit Implementation

The **orchestrator** runs the commit — all implementation code, tests, reviews, and SDD artifacts. No co-author attribution.

#### 4i. Completion Announcement

> Implementation complete! All requirements from SPEC-[###] have been implemented, reviewed, and tested.
> All artifacts committed. Feature is ready for deployment.

---

## Continuation Logic

When the user runs `/sdd-flow continue`:

1. Read `SDD/prompts/context-management/progress.md`
2. Determine which phase and sub-step is active
3. Resume from the exact sub-step where work was interrupted by spawning the appropriate subagent
4. If a phase was marked complete in progress.md, advance to the next phase

### Phase Detection Priority

- If "Implementation Phase - COMPLETE" → Done, show final summary
- If implementation is active → Resume the appropriate sub-step (4a-4h)
- If "Planning Phase - COMPLETE" → Start Step 4 (implementation)
- If planning is active → Resume the appropriate sub-step (3a-3e)
- If "Research Phase - COMPLETE" → Start Step 3 (planning)
- If research is active → Resume the appropriate sub-step (2a-2e)
- If no phase info → Start from Step 0 (scope assessment)

## Subagent Guidelines

### Prompt Construction

When spawning each subagent, the orchestrator must include in the prompt:
1. The full SDD command instructions for that step (model checks stripped)
2. All resolved artifact paths (inputs and outputs)
3. The task description and canonical identifiers
4. The project's CLAUDE.md instructions (if relevant to the phase)
5. An explicit instruction to read input files before starting work
6. An explicit instruction to create directories before writing output files

### Context Management Within Subagents

- Each subagent gets a fresh context window — no carryover from previous phases
- If a single subagent's task is too large, the orchestrator should split it into multiple sequential subagents
- Subagents should use Explore subagents (nested) for file discovery to preserve their own context
- Subagents should use general-purpose subagents (nested) for complex analysis tasks

### Error Handling

- If a subagent fails or returns incomplete results, the orchestrator should:
  1. Log the failure in `progress.md`
  2. Attempt to re-spawn the subagent with additional context about what went wrong
  3. If the same subagent fails twice, stop and inform the user

## Key Principles

1. **Each phase must be thorough** — don't rush through research to get to implementation
2. **Research informs planning, planning constrains implementation** — maintain this chain
3. **Every requirement gets a test** — no exceptions
4. **Reviews are mandatory, not optional** — critical review happens after every phase, code review happens during implementation
5. **ALL review findings must be resolved before proceeding** — every issue (HIGH, MEDIUM, LOW) gets fixed, not just noted. Reviews are gates, not checkboxes
6. **Document deviations** — if implementation diverges from spec, document why
7. **The spec is the source of truth** — implementation decisions trace back to spec requirements
8. **Never persist PII or secrets** in SDD documents
9. **Commit messages have NO co-author attribution** — per project convention
10. **Explicit paths always** — every subagent gets resolved, concrete file paths. Never rely on a subagent to guess or discover artifact locations.

## Arguments

| Argument | Description |
|----------|-------------|
| `<task>` | The task, requirement, or feature description to develop |
| `--auto` | Run in fully autonomous mode (no checkpoints) |
| `--supervised` | Run in supervised mode with checkpoints (default) |
| `continue` | Resume from the last interruption point |

## Examples

```
/sdd-flow Add GDPR-compliant audit logging for all anonymization requests
/sdd-flow --auto #15 Implement allow-list management UI with CRUD operations
/sdd-flow continue
```
