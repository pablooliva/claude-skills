# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code skills — each skill is a self-contained directory with a `SKILL.md` file that defines a structured, repeatable workflow invokable via slash commands.

## Structure

- Each skill lives in its own top-level directory (e.g., `sdd-flow/`, `improve-claude-md/`)
- The only required file per skill is `SKILL.md` with YAML frontmatter (`name`, `description`) followed by the skill's full instructions
- Skills are installed by copying/symlinking into `~/.claude/skills/`

## Conventions

- Commit messages should have NO co-author attribution (this is an explicit project convention)
- Skills are documentation-only — no build, lint, or test commands exist
- The README serves as the collection index; update it when adding or removing skills
