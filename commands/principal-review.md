# Principal Engineer Review

Inspect a codebase, carve it into logical chunks with minimal overlap, then fan out a panel of Principal-engineer subagents — one per chunk — to surface violations of DRY, KISS, SOLID, YAGNI, and Uncle Bob's Clean Code principles (small focused functions, intent-revealing names, Law of Demeter, code smells like rigidity / fragility / needless complexity, etc.). Aggregate every finding into a single markdown table the human can scan top-to-bottom.

The animating question for every reviewer is:

> Is this the best way we could have done this? Is it brittle? Could it be reusable, simpler, or better abstracted? Why was it done this way, and what is going to bite us later?

**A clean report is the goal, not the failure mode.** If the codebase is in good shape, the right answer is "no findings worth raising" — and that is a *more valuable* outcome than five fabricated nitpicks. A command that always finds something new on every run trains the human to ignore it; a command that only flags real issues earns the human's attention when it does. Treat "this codebase is in good shape" as a first-class result and present it confidently.

## Input

`$ARGUMENTS` — optional path to review. Defaults to the current working directory.

The path can be either:

- **A full repository root** — review the entire codebase end-to-end (chunks will likely split BE / FE / shared / etc.).
- **A subdirectory inside a repo** — review only that slice for a more granular pass (e.g. `src/server/billing/`, `web/components/forms/`, `packages/auth/`). The chunking step still applies: the subdirectory is itself decomposed into logical chunks based on what's inside it.

Treat the path as the **scope ceiling** — never reach outside it during the review. If the user passes a subdirectory, do not pull files from the broader repo into the chunk plan. Cross-directory Grep / Read by the subagents is allowed only for verification (e.g. "is this helper imported anywhere outside the scope?") — never to expand the review surface.

When the path is a subdirectory:

- The Codebase Map in the final report should describe **the scoped slice**, not the whole repo, but should note in one line which repo / parent project it lives in (e.g. "Reviewing `web/components/forms/` — a sub-module of a Next.js SaaS app").
- The chunking heuristics ("aim for 3–10 chunks", "shared code is its own chunk") still apply, but at the scale of the subdirectory. A 600-line subdirectory might cleanly review as 1–3 chunks; a 15,000-line subdirectory might still split into 5–8.
- If the subdirectory is so small that chunking adds no value (say, under ~500 LOC across a handful of files), skip the fan-out and run a single `review-principal` pass on the whole scope. Tell the user that's what you're doing and why.

## Process

You are the orchestrator. You do **not** review any code yourself. Your job is to (1) carve the codebase into clean logical chunks, (2) fan out the `review-principal` subagent across them, and (3) merge their findings into one table.

### Step 1: Map the codebase

Use an Explore subagent to walk the repository and produce a structural map. The map needs to capture, at minimum:

- The top-level layout (monorepo? single app? library?).
- The primary language(s) and framework(s) in use.
- The split between backend and frontend code (if any).
- The major directories and what they appear to contain (routes, components, utils, services, engine, schema, jobs, workers, etc.).
- File counts and rough line counts per major directory so chunk sizes are knowable.
- Anything obviously off-topic (generated artifacts, vendored deps, build output, fixtures) that should be excluded from review.

Skip `node_modules`, `vendor`, `.git`, `dist`, `build`, `.next`, `target`, `__pycache__`, and similar generated/vendored directories. Skip lockfiles, minified bundles, and snapshots.

If the path is empty or unreadable, tell the user and stop.

### Step 2: Carve the codebase into logical chunks

From the structural map, define a set of **logical chunks** — each chunk is a self-contained slice of the codebase that one Principal engineer can review in a single sitting and reason about coherently. Good chunks have **minimal overlap** with each other and **high internal cohesion**, so that DRY / SOLID violations surface within a chunk rather than getting lost across chunk boundaries.

Examples of good chunking shapes (these are illustrative, not prescriptive):

- **Backend** — `server functions / handlers`, `data-access layer`, `domain engine / business logic`, `background jobs / workers`, `shared utilities`, `schema / migrations`.
- **Frontend** — `routes / pages`, `reusable components`, `screen-level components`, `state management / stores`, `client-side utilities`, `hooks`.
- **Full-stack monorepo** — split BE and FE first, then chunk within each.
- **Library** — `public API surface`, `internal modules`, `tests / harnesses`, `examples`.

**Chunking heuristics:**

1. Aim for **3–10 chunks total**. Fewer than 3 means the codebase is too small to bother splitting (review it as one chunk); more than 10 usually means the chunks are too granular and findings will fragment.
2. Each chunk should be **roughly comparable in size** — if one chunk is 10× the others, split it further or rebalance.
3. Prefer chunks that match the codebase's own organisation. If the repo already has a `services/`, `routes/`, `components/`, `utils/` split, use it. Don't invent a new taxonomy when the existing one is sound.
4. **Shared / cross-cutting code is its own chunk.** A `shared/` or `common/` directory deserves its own reviewer because that's exactly where DRY decisions need to be evaluated.
5. If two candidate chunks have heavy overlap (the same files would appear in both), merge them — overlap defeats the purpose of fan-out.

Produce a chunk plan with:

| Chunk ID | Name                     | Paths included                 | Approx LOC | Why this chunk                                                               |
| -------- | ------------------------ | ------------------------------ | ---------- | ---------------------------------------------------------------------------- |
| C1       | BE — server handlers     | `src/server/handlers/**`       | 2,400      | All HTTP entry points, evaluable together for handler-level DRY/SRP          |
| C2       | BE — domain engine       | `src/engine/**`                | 3,100      | Pure business logic, evaluable for SOLID violations independent of transport |
| C3       | BE — shared utils        | `src/utils/**`, `src/lib/**`   | 850        | Cross-cutting helpers, prime DRY target                                      |
| C4       | FE — routes/pages        | `web/app/**`, `web/pages/**`   | 1,900      | Top-level UI orchestration                                                   |
| C5       | FE — reusable components | `web/components/**`            | 2,200      | Reusability evaluation                                                       |
| C6       | FE — screens             | `web/screens/**`               | 1,400      | Screen-level composition                                                     |
| C7       | FE — utils & hooks       | `web/utils/**`, `web/hooks/**` | 600        | Cross-cutting client logic                                                   |

Present this chunk plan to the user before spawning agents and ask whether they want to: (a) proceed as-is, (b) adjust the split, or (c) skip specific chunks. Use **AskUserQuestion** for this.

### Step 3: Fan out the `review-principal` subagent across chunks

For each chunk in the approved plan, spawn one `review-principal` subagent in parallel. Each gets a carefully scoped prompt containing:

- The **chunk ID and name** (so it can identify itself in the output).
- The **full list of paths** included in the chunk.
- A **short summary of the codebase as a whole** (one paragraph) so the reviewer understands the surrounding context — e.g. "This is a Next.js SaaS app with a Postgres backend; you are reviewing the FE reusable components chunk."
- The **language / framework / stack** signals from Step 1.
- An instruction to **read every file in the chunk** before forming findings.
- The standard Principal-engineer rubric (DRY / KISS / SOLID / YAGNI / Clean Code + the judgement questions) — this lives in the agent itself, so you don't need to repeat it in the prompt.

Spawn all subagents **simultaneously** via a single message with multiple Agent tool uses.

### Step 4: Aggregate findings into one table

Once all subagents return:

1. **Deduplicate cross-chunk findings.** If two reviewers flag the same shared utility from different angles, merge them into a single row and list both chunks in `Chunk(s) Flagged In`.
2. **Sort** by `Severity` (HIGH → MEDIUM → LOW), then by `Principle` (DRY / KISS / SOLID / YAGNI / Clean Code), then by chunk.
3. **Drop noise.** If a reviewer flagged something that contradicts another reviewer's finding (e.g. one says "extract this helper", another says "this is fine as-is"), surface the conflict in a `Conflicts` section below the table rather than dropping either side silently.

### Step 5: Present the final report

Output in exactly this structure:

```
## Principal Engineer Review

### Codebase Map
[2–4 sentence summary: stack, scale, high-level architecture]

### Chunks Reviewed
[List of chunks with one-line description each]

### Findings

| # | Severity | Principle | Chunk(s) | File(s) | Finding | Why It Matters | Recommendation |
|---|----------|-----------|----------|---------|---------|----------------|----------------|
| 1 | HIGH | DRY | C5 | web/components/FormField.tsx, web/components/EditField.tsx, web/components/InlineEdit.tsx | Three near-identical form-field components with copy-pasted validation and onChange wiring. | Bug fixes need to be applied in three places; visual drift is already happening between them. | Extract a dumb `<EditableField>` primitive that takes `value`, `onChange`, `validator`, and `renderInput`. Migrate all three call sites. |
| 2 | HIGH | SOLID (SRP) | C2 | src/engine/billing.ts | `processBilling()` charges the card, sends the receipt email, writes the audit log, and updates the analytics counter — four responsibilities in one 180-line function. | Any of those four concerns changes for unrelated reasons; tests have to set up all four to exercise the function. | Split into `chargeCard`, `sendReceipt`, `recordAuditEntry`, `incrementAnalytics`, orchestrated by a thin `processBilling` that composes them. |
| 3 | MEDIUM | KISS | C1 | src/server/handlers/search.ts | Custom hand-rolled query builder with 6 nested switch statements where the ORM's `where()` chain would do the same in 8 lines. | Reads as much more sophisticated than it needs to be; new contributors avoid touching it. | Replace with idiomatic ORM usage; the perf claim in the comment is unverified and the code is 4× longer. |
| 4 | MEDIUM | YAGNI | C2 | src/engine/plugin-registry.ts | Full plugin-loader scaffolding with hot-reload hooks, but only one plugin is ever registered and no roadmap mentions more. | Carrying complexity that doesn't earn its keep; every reader has to load the indirection into their head. | Inline the single plugin's logic into the engine; revive the registry only when a second plugin is on the table. |
| 5 | LOW | DRY | C3, C7 | src/utils/format-date.ts, web/utils/format-date.ts | Two implementations of `formatDate` diverging on Sunday-vs-Monday week start. | Subtle UI inconsistency between server-rendered and client-rendered dates. | Move to a shared package or keep one canonical implementation and import on both sides. |

### Conflicts
[List any places where two reviewers gave contradictory advice, with both views, so the human can adjudicate. Empty section if none.]

### What Each Reviewer Looked At
<details>
<summary>C1 — BE server handlers</summary>

[Full report from the C1 reviewer]

</details>

<details>
<summary>C2 — BE domain engine</summary>

[Full report from the C2 reviewer]

</details>

[... one per chunk ...]
```

**If no findings surfaced across any chunk, that is a successful run, not an empty one.** Say so explicitly, list the chunks reviewed, and state with confidence that the codebase looks healthy on the DRY / KISS / SOLID / YAGNI / Clean Code axes. Include each reviewer's "what I checked but did not flag" notes in the per-chunk `<details>` blocks so the human can see *what was considered*, not just *what was raised*. A confident "no findings" answer is the most valuable output this command can produce — never apologise for it, never weaken it with hedges like "everything looks mostly fine but here are some nits", and never invent findings to pad the table.

## Guardrails

- **Do not over-apply DRY.** A function that is "somewhat similar" to another, where unifying them would mean a complex generic signature or a forced abstraction, is **not** a DRY violation. The agent prompt covers this — but you, as orchestrator, should sanity-check the final table for findings that read as "DRY for DRY's sake" and demote or drop them.
- **Do not pad the table.** A short, sharp table of 5 real findings beats 30 nitpicks. If a reviewer returned weak findings, omit them from the table (but keep them in the `<details>` per-chunk report so nothing is lost). An empty table is a legitimate, valuable result — a command that always finds something on every run trains the human to ignore it; a command that confidently says "this codebase is in good shape" earns trust.
- **Stay descriptive, not prescriptive.** Each finding's `Recommendation` should describe a direction, not a finished refactor. The human running this command is the one who decides whether to act.
- **Don't double-flag.** A naming or function-length issue that already shows up under Clean Code shouldn't be re-listed under KISS just to inflate the count. Pick the principle that fits best and put it there once.
- **No code modifications.** This command is read-only. Do not edit, format, or rename anything. The output is a report.

## Notes

- Large codebases: if the structural map suggests the codebase is over ~50,000 LOC, ask the user whether they want to scope the review to a subdirectory before chunking. A 50k-line review will produce more findings than a human can usefully act on in one pass. Re-running the command with a narrower path argument (e.g. just `src/billing/`) is the right follow-up move.
- Re-running this command on the same codebase a few weeks later is a healthy practice — the chunk plan should be stable, and the diff between findings tells you whether the team is closing or opening tech-debt issues. A run that goes from 8 findings to 0 is the success state this command exists to confirm; resist any instinct to "find something new" just because the previous run had a longer table.
