# Slim CLAUDE.md

Audit the project's CLAUDE.md and rewrite it to contain **only knowledge Claude can't infer by reading the code**. Every byte of CLAUDE.md costs context tokens on every message, so file trees, script lists, and per-file descriptions of well-documented functions are tax, not signal.

## Input

`$ARGUMENTS` — optional path to a CLAUDE.md file. If omitted, defaults to `./CLAUDE.md` at the repo root.

## Guiding principle (Anthropic's own advice)

Focus on what Claude **cannot infer** from reading the code.

**Keep** (these have no analogue elsewhere):

- Non-obvious policy and team conventions ("we never inline math into components", "use Bun not Node")
- Historical context for hard-won lessons (past incidents, the *why* behind a defensive guard)
- Things Claude repeatedly gets wrong without help ("don't use the `pg` package — go through Prisma")
- Invariants that aren't in code comments or schemas
- Architectural framings that span files and aren't obvious from any single one
- "North star" / philosophy statements that drive judgement calls

**Cut** (recoverable in seconds via `ls`, `grep`, or `cat package.json`):

- File listings or directory trees
- Lists of scripts that mirror `package.json`
- Per-file bullets describing functions that already have header comments
- Verbose explanations of standard tools (Claude already knows how React, Express, Prisma work)
- Stack lists that duplicate dependencies
- Things that change frequently (commit hashes, version numbers)

**Lazy-load** (don't inline — reference with `@`):

Inside CLAUDE.md you can write `@path/to/file.md` and Claude will only load that file when its content becomes relevant. This is the right tool for material that is *important but not needed on every message* — testing conventions, contribution guides, architecture deep-dives, runbooks, design docs, ADRs. Inline the irreplaceable 1-pager in CLAUDE.md itself; push everything else into linked files (`@TESTING.md`, `@docs/architecture.md`, `@CONTRIBUTING.md`, etc.) and reference them by path. The reference itself costs a handful of tokens; the file costs nothing until Claude opens it.

Rule of thumb: if a section is read every few messages → inline it. If it's read once a week → make it a separate file and `@`-link it. If the file already exists, replace any duplicated content in CLAUDE.md with the `@`-reference and a one-line hint about when to open it.

## Process

You are the orchestrator. Follow this exact workflow — it's proven to land a ~50% cut without losing load-bearing knowledge.

### Step 1: Read the current CLAUDE.md

Read the file at the input path. If it doesn't exist, tell the user and stop.

### Step 2: Audit each claim against the code

Launch an **Explore** subagent in parallel with reading. Give it the list of claims from CLAUDE.md and ask it to classify each as YES (already self-evident in the code — comments, header docs, schemas, lint rules, `package.json`, etc.) or NO (CLAUDE.md is the sole source). Also ask it to identify any sibling docs (`TESTING.md`, `CONTRIBUTING.md`, `docs/*.md`, `ADRs/`, `RUNBOOK.md`, etc.) and whether CLAUDE.md duplicates content from them — those are prime `@`-reference candidates.

Tailor the audit checklist to the actual contents of the file. Typical checks:

1. Does each "server-only" / `"use server"` / similar file-list claim hold up? (spot-check 5 random files start with the claimed marker)
2. Do described data flows have any code comments, or is CLAUDE.md the only narrative?
3. Do per-file business-logic descriptions duplicate existing JSDoc / header comments?
4. Does the schema (Prisma / SQL / OpenAPI) have inline comments for the invariants CLAUDE.md describes?
5. Are described lint rules / boundaries enforced by tooling (ESLint config, etc.)?
6. Is the philosophy / "north star" section in code comments, or only here?
7. Is incident background (e.g. "on YYYY-MM-DD we lost data because…") preserved in any code comment near the guard?
8. Are package scripts duplicated from `package.json`?
9. Is the Project Structure tree just `find src/ -type f` output?
10. Are listed actions / routes recoverable by `ls`?
11. Is the lint-suppression / contribution policy enforced by tooling or sole-source?
12. Is the pre-commit hook config visible in `.husky/` or `lint-staged.config.js`?

Ask for a YES/NO list and a short closing note on what's surprising.

### Step 3: Ask the user how aggressive to be

Use AskUserQuestion with three options (preview-style):

- **Aggressive** — cut all duplication, target ~50% reduction. Drop the directory tree, scripts list, per-file business-logic bullets, per-page architecture walkthroughs, and any other duplication. Keep only the irreplaceable.
- **Moderate** — keep page-level data-flow narratives and a condensed per-file block; drop only the most obvious duplication (Project Structure tree, Scripts list, server-only file list).
- **Conservative** — drop only the directory tree, scripts list, and server-only file list. Leave everything else intact.

Show what's removed vs kept in the preview for each. Default to recommending **Aggressive**.

### Step 4: Ask whether to backport schema invariants

If the audit found that the data-model section is the sole source for invariants not in the schema file, ask whether to also mirror those invariants as inline comments in the schema (e.g. `prisma/schema.prisma`). Recommend **No** unless the user wants the extra durability — it expands scope and creates a sync burden.

### Step 5: Write a plan file (only if in plan mode)

If `plan mode` is active, write the audit + slim-down plan to the plan file and call `ExitPlanMode`. Otherwise skip to Step 6.

### Step 6: Rewrite CLAUDE.md in place

Use `Write` to fully replace CLAUDE.md. **Before writing, replace duplicated content with `@`-references** to existing sibling docs whenever possible (e.g. `See @TESTING.md for the full testing conventions.` rather than re-summarising them). If a section is large and could stand alone, consider extracting it to a new file under `docs/` and `@`-linking it — but only do this if the user agreed to it in Step 3 / 4 or if it's obviously a separate concern.

Structure the new file roughly as:

1. Non-obvious policy / philosophy (BDD flow, north star, etc.) — if present in original
2. IP / Safety / DB rules — if present
3. Historical incident narratives + structural guards
4. Architecture in 3–5 lines stating only load-bearing invariants
5. What the app does in 2–3 lines (skip if not in original)
6. Data-model invariants **not** documented in the schema
7. Tooling-enforced boundaries with their rationale
8. Linting / contribution policy
9. Testing one-liners + the non-obvious traps

Verify each load-bearing claim from the original survives — explicitly check off:

- All numbered policy rules
- Each incident narrative
- Each "north star" / philosophy paragraph
- Each schema-absent invariant
- Each "things Claude gets wrong" rule
- The non-obvious commands and traps (e.g. "use `test:all` not `test`")

### Step 7: Report

Run `wc -l` and `wc -c` before/after. Report the cut percentage. List what was removed and confirm where it's recoverable (one line each). End-of-turn summary: one sentence on what changed.

## Notes

- Don't commit unless the user asks. This is a one-file rewrite; the user will review the diff.
- If the user pushes back on what was cut, the audit checklist tells you exactly where the original claim came from — you can grep for it and restore.
- Tone of the new CLAUDE.md should match the project's existing voice — terse for terse, formal for formal. Don't impose a house style.
- The plan file is your scratchpad; do not write secondary files (no `REMOVED.md`, no audit report).
