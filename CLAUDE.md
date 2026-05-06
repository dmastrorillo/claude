# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Personal store for Claude Code customisations: hand-authored slash commands, subagents, and skills, plus a lockfile of third-party skills worth keeping. Nothing here builds, runs, or tests — every artifact is a Markdown file (sometimes with YAML frontmatter) consumed by the Claude Code harness when copied/symlinked into `~/.claude/` or a project's `.claude/`.

## Layout

- `commands/*.md` — custom slash commands. Filename (minus `.md`) becomes the command name (`commands/spar.md` → `/spar`). Body is the prompt; optional YAML frontmatter sets `description`. `$ARGUMENTS` is the standard placeholder for user input.
- `agents/*.md` — subagent definitions. Each file requires frontmatter with `name`, `description`, `tools`, and `model` (typically `sonnet`).
- `skills/<skill-name>/SKILL.md` — custom skills. Folder name = skill name. Frontmatter requires `name` and `description` (the description is the trigger surface, so it must be specific about *when* to invoke).
- `skills-lock.json` — artifact for the `skills` npm package (https://www.npmjs.com/package/skills). Used to record third-party skills to re-install on a new machine. Add new entries by following the existing shape exactly — the format is what the `skills` tool consumes, so deviations will break installs.
- `README.md` — currently empty.

## Working in this repo

- When adding a command, agent, or skill, mirror the conventions of the existing files in that directory rather than inventing new frontmatter shapes — the harness is strict about field names.
- The `commands/review.md` orchestrator delegates to the four `review-*` subagents in `agents/`. If you rename or move one, update the other.
- When adding to `skills-lock.json`, copy the exact field shape of nearby entries — the `skills` npm tool relies on it.
- These files are reference material, not executed code: prefer minimal, surgical edits over rewrites, and don't add tests, linters, or build tooling.
