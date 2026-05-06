---
name: change-request
description: Three-phase workflow for any change to the budgeting app — align on BDD scenarios in test-cases.md first, then drive tests via the tdd skill, then change production code in vertical RED/GREEN slices. Use when the user asks to add a feature, fix a bug, refactor behaviour, or tweak UX — anything with observable consequences. Also trigger on 'change request', mentions of test-cases.md, BDD, Given/When/Then scenarios, the 'shape the app' workflow, or when the user says 'let's work on…' / 'I want to add…' / 'fix the bug where…' against this codebase. Keeps test-cases.md authoritative so the codebase moves where the cases point.
---

# Change Request Workflow

Every change to this app — feature, bug fix, refactor with observable consequences, UX tweak — flows through three phases in order: **BDD → tests → code**. The phases exist to keep `test-cases.md` (the behavioural spec at the repo root) authoritative. The app moves in the direction the test cases point, not the other way around.

The first phase is the one most often skipped. Don't skip it. If the BDD isn't agreed before tests are written, the work drifts and the spec rots.

## The flow

```
Phase 1: BDD          → open test-cases.md, draft/update Given/When/Then,
                        get user alignment on the behaviour
                              ↓
Phase 2: Tests        → invoke the tdd skill, write RED tests that
                        assert the agreed cases, tagged with TC-IDs
                              ↓
Phase 3: Code (GREEN) → smallest vertical slice that turns the RED
                        tests GREEN, refactor only after green,
                        no scope creep
```

Each phase is gated on the one before. Don't write tests before BDD is agreed; don't write code before tests are RED.

## Phase 1 — BDD alignment

Open `test-cases.md` and find the section that matches the change. Section/category map is in the table of contents at the top of the file.

Decide which kind of change this is:

| Kind                                         | What to do in test-cases.md                                                                                                                                                                                                                                   |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New feature / new capability                 | Append new cases at the next unused number in the relevant category. Use the next-number rule per CLAUDE.md (zero-padded `TC-<CATEGORY>-<NUMBER>`).                                                                                                           |
| New edge case discovered                     | Append at the next unused number in the relevant category.                                                                                                                                                                                                    |
| Behavioural change to existing functionality | Edit the affected cases in place. Don't leave stale wording. If the change is large enough that one case becomes two, use the case-splitting rule from CLAUDE.md (keep the original ID for the dominant assertion, new sub-case gets the next unused number). |
| Bug fix                                      | Make sure the primary case in the relevant section describes the fixed behaviour. Add a regression entry in section 24 cross-referencing the primary case (one-line "See TC-XXX-NNN" plus a sentence of incident context).                                    |
| Pure refactor with no observable change      | No spec change. Skip to phase 2.                                                                                                                                                                                                                              |

Draft the Given / When / Then in the user's voice — what do they see, what do they do, what is the observable outcome. Avoid implementation-level language (function names, field names, DB terms) in sections 1–20 and 24. Sections 21–22 and a few VAL cases in 23 are explicit developer-process exceptions.

**Then propose the draft cases to the user and wait for alignment.** This is the gate. Phase 2 starts only after the user agrees the cases describe the behaviour they want.

If the user redirects ("no, that's not quite right…"), update the cases and re-propose. The conversation in this phase is about what the app should DO, not how to build it.

## Phase 2 — Tests via the tdd skill

Once the BDD is agreed, invoke the `tdd` skill (`/tdd`) to drive the red-green-refactor loop. This skill handles the test/code mechanics — RED first, then minimum code to GREEN, then refactor.

Two things to carry forward from phase 1 into TDD:

1. **TC-IDs in test descriptions.** Every new test name leads with the relevant case ID, e.g. `test("TC-PAID-001 — marks an expense occurrence as paid")`. This makes traceability between spec and code automatic. Per CLAUDE.md, existing pre-spec tests are exempt until next touched; new tests are not.

2. **Test tier per case.** Use TESTING.md's triage table to pick the layer (unit / property / component / integration / E2E). Cases tagged `(property invariant)` in `test-cases.md` belong in `*.property.test.ts` files using fast-check.

Hand the tdd skill the specific TC-IDs to drive against. Don't broaden scope beyond those cases.

## Phase 3 — Code in vertical slices (GREEN)

The tdd skill handles this — it's a thin wrapper around the red-green-refactor loop. The only thing this workflow adds:

- **Vertical slices.** Each slice is the smallest end-to-end change that turns one or more RED tests GREEN. Don't write half a UI then half an engine — pick one observable outcome (a TC case), drive it red-then-green, commit-shaped, move on.
- **No scope creep.** If you spot something else broken or unclear, don't fix it inline. Either add a TC case for it (loop back to phase 1) or note it for a follow-up.
- **Refactor only after green.** Improvements to internal shape happen on green tests, not while the suite is red.

## Mid-implementation clarifications

It's normal to discover a case is ambiguous or incomplete during phase 2 or 3. When that happens, **loop back to phase 1**: update the case in `test-cases.md` as part of the same change. Don't code around the ambiguity. Don't silently deviate from the spec. The relief valve is documented in CLAUDE.md and is part of the workflow, not an exception to it.

If the clarification changes the user-observable behaviour (not just wording), pause and confirm with the user before continuing.

## Before merging

Run through this gate:

- [ ] Every TC case touched by this change reads as truth — no stale wording, no contradictions.
- [ ] Every new test description includes its TC-ID.
- [ ] The full test suite passes (unit + property + component + integration; E2E if a user-facing flow changed). See TESTING.md for the "which command when" table.
- [ ] If a case was deliberately retired, it has a tombstone comment per CLAUDE.md's retirement rule.
- [ ] If the change fixes a bug, section 24 has an entry pointing at the primary case.

## What this skill does NOT do

- It does not perform tests or code changes itself — phase 2 hands off to the `tdd` skill.
- It does not gate code review or architecture decisions — those are separate concerns.
- It does not replace user judgement on what the app should do — phase 1 is a conversation, not a unilateral decision.

## Quick-reference example

User asks: _"Can we make the dashboard refresh every 30 seconds?"_

1. **Phase 1.** Open `test-cases.md` section 8 (DASH). No case currently covers auto-refresh. Draft TC-DASH-010 ("Dashboard refreshes every 30 seconds without user action — Given/When/Then…"). Propose to user. They might redirect ("only when the tab is focused") — update the draft. Once agreed, the case is final.
2. **Phase 2.** Invoke `/tdd`. Write a RED test like `test("TC-DASH-010 — dashboard re-fetches every 30 seconds when tab is focused")`. Confirm it fails for the right reason.
3. **Phase 3.** Smallest change to turn it GREEN. Run the suite. Refactor if needed on green.
4. **Pre-merge.** TC-DASH-010 reads as truth, test description includes the ID, suite passes.
