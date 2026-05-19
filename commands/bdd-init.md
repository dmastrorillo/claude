# /bdd-init — Bootstrap the BDD/TDD workflow in a project

Stand up the **OpenSpec → BDD → TDD → code → user-visible behaviour** pipeline in a project (or in a feature sub-directory inside a larger project). This command creates the `test-cases.md` behavioural-spec contract, the `TESTING.md` conventions doc, and updates `CLAUDE.md` to point at both — so every future change in scope flows through the same disciplined loop.

The philosophy this command installs:

> **`test-cases.md` is the contract; the code is downstream.** A change is "real" only after it appears as a Given/When/Then in `test-cases.md`, is exercised by a test that names its TC-ID, and is implemented behind that test. The north star is **what the user observes** — never the value of an internal struct, the shape of a returned object, or the math an engine function returned.

This is not a one-time scaffold. The files this command creates become permanent fixtures of the project: humans and Claude open `test-cases.md` first on every change, write tests that cite its TC-IDs, and only then write production code.

---

## Input

`$ARGUMENTS` — optional directory where the spec scope lives. Defaults to the repo root. Pass a feature sub-directory (e.g. `shell/src/features/ai-admin`) to scope BDD to that feature only, leaving the rest of the project untouched.

---

## Guiding principle

Follow Anthropic's advice for `CLAUDE.md`: keep only what Claude cannot infer from code. The workflow narrative, north-star philosophy, TC-ID scheme, layer-triage table, mutation-testing rules — these live in `TESTING.md` (lazy-loaded via `@TESTING.md`) so they cost zero tokens until relevant. `CLAUDE.md` carries only a one-line workflow summary + the `@TESTING.md` pointer + whatever project-specific context already exists in the file.

`test-cases.md` is hand-edited as part of the same change that proposes new behaviour — it is **not** a generated artifact. There is no script. The "generation" is the human-with-Claude translation step from an OpenSpec proposal into Given/When/Then scenarios.

---

## What this command creates

1. **`<scope>/test-cases.md`** — the BDD behavioural spec. Starts with a header that names it as the contract, a Categories table seeded with the user's chosen codes, and one example case so the format is unmistakable.
2. **`<scope>/TESTING.md`** — testing conventions: north star, TC-ID scheme, TC-ID tagging rule, layer triage (unit / property / component / integration / Playwright E2E), property-test guidance (fast-check), mutation-testing notes (Stryker or equivalent), TDD loop, regressions policy.
3. **`<scope>/CLAUDE.md`** — created if missing; **updated** (not overwritten) if present. Adds a single "Workflow (mandatory)" section with the one-line flow and the `@TESTING.md` reference. Leaves everything else in the file alone.

---

## Process

You are the orchestrator. Do not skip steps — each one prevents a specific class of botched scaffold.

### Step 1: Resolve the scope

Read `$ARGUMENTS`. If empty, the scope is the repo root (working directory). Verify the path exists. If it doesn't, ask the user whether to create it.

Check what already exists in scope:

- Does `test-cases.md` exist? If so, **stop and ask** — the user is either re-running the command (offer to skip), starting fresh (offer to back up the existing file as `test-cases.md.bak-<date>`), or trying to extend a partial scaffold (offer to merge by adding only missing categories).
- Does `TESTING.md` exist? Same three-option question.
- Does `CLAUDE.md` exist? If yes, you will append, not overwrite.

### Step 2: Detect project context

Use the Explore subagent (or quick `Bash` + `Read`) to identify:

- **Language / test runner.** Look for `package.json` (Node/Bun/Yarn), `go.mod` (Go), `pyproject.toml` / `requirements.txt` (Python), `Cargo.toml` (Rust), `composer.json` (PHP), etc. Read the `scripts` / `[scripts]` / `Makefile` to spot the actual test commands.
- **Existing test layout.** `**/*.test.*`, `**/*.spec.*`, `tests/`, `e2e/`, `cypress/`, `playwright.config.*`, `vitest.config.*`, `jest.config.*`, `stryker.conf.*`, `fast-check` in dependencies.
- **Existing OpenSpec dir.** `openspec/` at the repo root.
- **Project type signals.** React / React Native / Next.js / Vue / pure-library / CLI / backend service — these drive _which testing layers actually apply_ (a Go CLI has no component tests; a React app has no `go test`).

You will tailor the generated `TESTING.md` to the layers that exist or could plausibly be added. Do not generate a Playwright section in a Go CLI scaffold.

### Step 3: Ask the user the four scaffold questions

Use **one** `AskUserQuestion` call with up to four questions. Don't drip them.

1. **Initial categories for `test-cases.md`.** Suggest 3–5 starter categories inferred from the project (e.g. for a CRM feature: `AUTH`, `TASK`, `MSG`, `UPL`, `REG`; for a CLI: `CMD`, `CFG`, `ERR`, `REG`; for a budgeting app: `TXN`, `PAID`, `OVERDUE`, `ALLOC`, `REG`). Categories are stable codes — they're hard to rename later, so they're worth deciding now.
2. **OpenSpec.** Does the project already use OpenSpec? If not, offer to scaffold it. Default: **Yes, scaffold** if no `openspec/` exists at the repo root using `npx openspec init`
3. **Mutation testing.** Is mutation testing in scope for this project? Stryker for JS/TS, `go-mutesting` for Go, `mutmut` for Python, `cargo mutants` for Rust. Default: **Yes, document it** — the discipline matters even if the tool is added later.
4. **E2E layer.** Playwright, Cypress, or none? Default: **Playwright** for browser apps, **none** for libraries / CLIs / backend services where unit + integration suffice.

### Step 4: Write `test-cases.md`

Use the template below. Fill in the Categories table with the user's choices from Step 3. Seed one example case per category so the format is unmistakable. Tag each example case with `<!-- example: delete when real cases land -->` so it's clear the seeded cases are not load-bearing.

```markdown
# test-cases.md — <project / feature name> behavioural specification

This file is the authoritative, human-readable specification of how <project> behaves. It holds BDD-style Given/When/Then scenarios covering happy paths, edge cases, and known historical regressions. **It is the contract; the code is downstream.**

The flow is: **OpenSpec proposal → BDD cases here → tests → production code → observed behaviour.** A change is "real" only after it appears as a Given/When/Then below, is exercised by a test that names its TC-ID, and is implemented behind that test.

See [`TESTING.md`](./TESTING.md) for the full pipeline, TC-ID rules, and layer triage.

---

## Categories

Each case has an ID of the form `TC-<CATEGORY>-<NUMBER>`. Categories are short, stable codes; numbers increment within each category starting at `001`, zero-padded to 3 digits. **Never renumber existing IDs.**

| Code            | Scope                                                 |
| --------------- | ----------------------------------------------------- |
| [`CAT1`](#cat1) | <one-line scope>                                      |
| [`CAT2`](#cat2) | <one-line scope>                                      |
| [`REG`](#reg)   | Explicit dated regression tests for shipped incidents |

---

## CAT1

### TC-CAT1-001 — <one-line summary> <!-- example: delete when real cases land -->

- **Given** <user-centric precondition>,
- **When** <user action / observable event>,
- **Then** <user-observable outcome>.

Exercised by `<path/to/test/file>` → `<test name including TC-CAT1-001>`.

---

## CAT2

### TC-CAT2-001 — <one-line summary> <!-- example: delete when real cases land -->

...

---

## REG

(Dated regression cases land here, each cross-referenced to the original case it shadows.)
```

### Step 5: Write `TESTING.md`

Use the template below. **Prune sections that don't apply to the project** — drop the Playwright section if the user picked "none" in Step 3, drop the property-test section if the project has no candidate for invariant testing (rare), drop the mutation section only if the user explicitly declined it.

````markdown
# Testing — <project / feature name>

This document is the source of truth for testing conventions in this scope. It is referenced from `CLAUDE.md`. Keep both in sync when the rules here change.

---

## BDD spec is the contract

Before authoring any test, consult [`test-cases.md`](./test-cases.md) — the BDD spec is the source of truth for behaviour. The mandatory workflow is:

1. **OpenSpec proposal** describing the user-visible contract this change introduces or modifies. Behaviour decisions land here before any test or code.
2. **Translate the proposal into BDD cases in `test-cases.md`** — a hand-driven, AI-assisted edit, **not** a generated artifact. New feature / new edge case → add fresh `TC-<CATEGORY>-NNN` entries. Bug fix → add a regression case cross-referenced to the relevant section. Behavioural change → update affected cases in place; never leave stale scenarios.
3. **Invoke `/tdd` and write red/green slices.** Each case becomes a failing test that names its TC-ID in the description. Red → green → refactor.
4. **Implement until tests pass** and no existing tests regress.
5. **Before merging,** re-read touched cases and confirm `test-cases.md` still describes reality.

If a case and the code disagree, **the case is the spec** — investigate the code. The only exception is when the current task's explicit goal is to change that behaviour.

**Mid-implementation clarifications:** if a case is ambiguous while building, update the case in `test-cases.md` as part of the same change — never code around the ambiguity silently.

---

## The north star is what the user observes

Behaviour is what the user observes — rendered text, the rendered DOM, the bubble that appears after a click, the toast on failure, the exit code, the bytes on stdout, the file written to disk. It is **NOT** the value of an internal struct field, the shape of a returned object, or the math a function returned. Those are code-internal facts; they may or may not surface.

When a test claims a TC-ID, it MUST drive at the layer where the user observation happens. **Triage question for every TC-ID-tagged test:** "Could this test pass while the user sees something the spec forbids?" If yes, the spec is **under-tested** — add another test at the user-observable layer until the answer becomes "no."

Engine and helper tests are valuable scaffolding, but they NEVER satisfy a TC about what the user sees.

---

## TC-ID scheme

Format: `TC-<CATEGORY>-<NUMBER>`. Category codes live in the Categories table of `test-cases.md`. Numbers increment per category from `001`, zero-padded to 3 digits.

- New case in an existing category → next unused number.
- New category → add the code to the ToC, start at `001`.
- **Never renumber existing IDs** — tests reference them.
- Each case is self-contained — reads without surrounding context.

**Case retirement / splitting:**

- Removed because feature was removed → leave a one-line tombstone (e.g. `<!-- TC-CAT-006 retired YYYY-MM-DD: feature dropped in commit abc123 -->`). Update or delete referencing tests.
- Split into two → keep the original ID for the dominant assertion, assign the new sub-case the next unused number, update referencing tests.
- Never silently delete a case whose TC-ID appears in test code.

---

## TC-ID tagging on every test

Every test (unit, property, component, integration, Playwright/E2E, mutation harness) that asserts a BDD scenario MUST cite its TC-ID in the test description:

```ts
test("TC-MSG-003 — Send is disabled when the composer is empty", () => { ... });
test("TC-AUTH-007 — invalid token logs the user out", () => { ... });
```
````

When one case is exercised by multiple tests at different layers (e.g. one Playwright happy-path + two unit tests for edge cases), **every test cites the same TC-ID**. A reader greps for the ID and finds every test that asserts the case. A test asserting genuinely new behaviour without a TC-ID is a defect — fix it by adding the case before merging.

---

## Test-level triage

Decide where a test belongs by the narrowest level that can reproduce the regression you want to catch.

| Level                     | Scope                                                     | When                                                                                         |
| ------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Unit**                  | A pure function or small module                           | Input/output logic, helpers, formatters, parsers, coercers                                   |
| **Property** (fast-check) | Pure-function invariants over generated inputs            | Any invariant that should hold for _every_ valid input, not just the examples you thought of |
| **Component**             | One UI component / tightly-coupled pair                   | Props-to-DOM, conditional rendering, user interaction on a single component                  |
| **Integration**           | Multiple modules wired together, or service + DB + engine | Data-integrity, ownership checks, cross-module wiring                                        |
| **E2E** (Playwright)      | Real browser + real server (or close to it)               | Happy-path journeys, auth + cross-page flows, irreversible data actions                      |

**Default bias:** push lower-value coverage to component + integration so the E2E suite stays fast and trustworthy. Do not reach for E2E just because a bug happened in the UI — if the bug is reproducible at a single component or a single service call, test it there.

---

## Property-based testing (fast-check)

Property tests live as `*.property.test.*` alongside their subject. They use `fast-check` to generate arbitrary inputs and assert invariants that must hold across the input space.

**When to add a property test:** when "for all valid inputs of shape X, Y holds" is the real statement. Don't enumerate examples when the invariant is general.

**Common shapes that earn property tests:**

- **Idempotence.** `f(f(x)) === f(x)`.
- **Monotonicity.** "When input parameter rises, output never falls."
- **Reordering commutativity.** "Sorting the input list does not change the output."
- **Reference cross-check.** Function under test vs hand-written reference formula.
- **Field-name reconciliation.** "For any doc with field X spelled either way, the normalised view's X is populated."
- **Never-negative / never-impossible.** Stress an output invariant across the input space.

Default `numRuns` is 100; raise only if coverage demands it. Each new property earns an entry in an invariant catalogue at the top of the property-test file (or in `TESTING.md`) so a reader can audit the invariant set without opening every file.

---

## E2E testing (Playwright)

<!-- delete this section if the project has no E2E layer -->

E2E tests live under `e2e/specs/*.spec.ts` (or equivalent). Playwright boots a real-ish stack and runs Chromium against it.

**When to add an E2E spec:**

- Happy-path user journeys that span multiple pages / Server Actions / external boundaries.
- Auth or cross-page flows where component + integration can't give confidence.
- Irreversible data actions (bulk submit, destructive delete).

Not a closed list. Before reaching for E2E, ask: **can this be reproduced at component or integration level?** If yes, test it there — E2E is the most expensive layer to write and to run.

**Infrastructure principles:**

- Isolated build / port / DB. E2E must never collide with `dev` server or touch the dev DB.
- Seeded fresh fixture per run.
- `storageState`-based auth where possible — don't drive the login form in every test.
- Single-worker / non-parallel by default — race conditions over a shared DB / dev server are not worth debugging.

---

## Mutation testing

<!-- delete this section if mutation testing is out of scope -->

Mutation testing audits the **diagnostic power** of the test suite. Line coverage tells you lines ran; mutation score tells you whether a test would fail if the logic under it changed subtly. **The two are not substitutes.** A 100%-line-covered file with a 30% mutation score has tests that _execute_ the code but don't _assert_ against it.

Tooling: **Stryker** for JS/TS, **`go-mutesting`** for Go, **mutmut** for Python, **`cargo mutants`** for Rust.

**Rules:**

- Mutation score **must not regress** per file. When migrating or rewriting tests, the replacement must match or exceed the prior file's score.
- When a mutant survives, **add a killing test** — do not add a disable comment.
- Run mutation testing scoped to a single file during iterative work; full-suite runs are minutes-long and are for milestone reviews.
- Mutation testing does not replace BDD-driven coverage — it's a second axis. A file can have a high mutation score on internal math while still failing the "user observable" triage question. Both layers matter.

---

## TDD workflow

Use the `/tdd` skill for every implementation slice. The loop is:

1. **Red.** Write the smallest failing test that names the TC-ID and asserts the user-observable behaviour. Run it; confirm it fails for the right reason.
2. **Green.** Write the minimum production code to make it pass. No speculative abstractions.
3. **Refactor.** Tidy with the test green.
4. **Repeat** for the next slice / case.

Do not write production code without a red test pointing at it. Do not write a test without a TC-ID. Do not write a TC-ID without an OpenSpec proposal it traces back to (unless the openspec proposal was skipped for a trivial change).

**Failing tests are regressions, not broken tests.** If a test fails, assume the code under test has regressed. Investigate the behaviour the test describes and fix the production code. Only rewrite a test's expectations when the specific behaviour it covers was deliberately changed, and that change is the actual goal of the current task. Never silence, skip, or loosen a failing test to make the suite green.

---

## Regressions

When fixing a bug:

1. Find the existing TC-ID the buggy code violates, if any. If found, the regression test reuses that ID — the bug was a coverage gap inside an existing case. Add the additional test exercising the previously-uncovered angle.
2. If the bug exposes genuinely new behaviour that had never been articulated, add a fresh `TC-<CAT>-NNN` case in the appropriate category — **not** under `REG`. `REG` is reserved for explicit dated "this exact incident must not return" tests; those are added in addition to the normal case, not in place of it.
3. Either way, the test goes RED first against the buggy code, then GREEN after the fix. No exceptions.

````

### Step 6: Update `CLAUDE.md`

If `CLAUDE.md` doesn't exist in the scope, create it with **only** the workflow section below. If it exists, **append** the workflow section after any existing front-matter / header content — do not rewrite the rest of the file.

```markdown
---

## Workflow (mandatory)

**OpenSpec proposal → BDD cases in `test-cases.md` → TDD via `/tdd` → code → user-visible functionality.** A change is "real" only when it appears as a Given/When/Then in `test-cases.md`, is exercised by a test that names its TC-ID, and is implemented behind that test.

See @TESTING.md for the full workflow, TC-ID scheme, layer triage, property-test guidance, mutation-testing rules, and stubbing conventions. Open it before writing any test or editing the spec.
````

### Step 7: (Optional) Scaffold OpenSpec

If the user answered "yes" to scaffolding OpenSpec in Step 3, use `npx openspec init` to scaffold OpenSpec at the repo root. This creates the `openspec/` directory with the default OpenSpec files.

If OpenSpec already exists at the repo root or the user answered "no" to scaffolding, do nothing.

### Step 8: Report

Run `wc -l` and `wc -c` on the created / updated files. Output a short report:

- Files created: list paths + sizes.
- Files updated: list paths + diff summary (e.g. "appended Workflow section, 6 lines").
- Categories seeded: the ones the user chose.
- Pruned sections: any TESTING.md sections that were dropped for this project type.
- Suggested next step: "Open an OpenSpec proposal for the first real change, then translate it into BDD cases in `test-cases.md`."

End-of-turn summary: one sentence on what was bootstrapped and where.

---

## Notes

- **Do not run any test or build commands.** This command bootstraps documentation; verifying that the test runner picks up files is the user's first real BDD task, not part of the scaffold.
- **Do not delete user content.** If `CLAUDE.md` / `TESTING.md` / `test-cases.md` exist, ask before any destructive operation.
- **Tailor to project type.** A Go CLI scaffold drops Playwright. A pure-library scaffold drops E2E entirely. A backend service drops component. The four AskUserQuestion answers in Step 3 are the primary tailoring signal; project-context detection in Step 2 is the secondary signal.
- **Prefer feature scope over repo scope when the project is large.** If `$ARGUMENTS` points at a sub-directory of a monorepo, scope everything there — the repo-root `CLAUDE.md` need not be touched.
- **Tone of the generated files should match the project's existing voice.** If the repo already has terse, opinionated docs, match that. If it has formal, lengthy ones, match that. Do not impose a house style.
- **Do not commit.** This is a multi-file scaffold; the user will review the diff before staging.
