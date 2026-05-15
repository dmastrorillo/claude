---
name: northstar
description: Defines and protects a product's north star. Use this skill whenever the user wants to (a) articulate what the product is trying to solve and capture it as docs/NORTHSTAR.md, or (b) decide whether a proposed feature aligns with the existing north star or represents abstraction/scope creep. Triggers on phrases like "north star", "northstar", "NORTHSTAR.md", "what is this product for", "create a product vision", "should we build this", "does this fit the product", "is this scope creep", "abstraction creep", "is this feature aligned", "should this be in scope". If docs/NORTHSTAR.md is absent, run a sparring session and write it. If it exists, treat the user's message as a feature proposal, stress-test alignment via a second sparring session, then recommend include / exclude / update-northstar — user makes the final call.
---

# Northstar

A `docs/NORTHSTAR.md` is a single page that says, in the user's own voice, what this product is trying to solve, for whom, and what is explicitly out of scope. It's the doc you point at when a teammate says "should we build X?" and the answer falls out of reading it.

This skill has two modes and picks between them automatically based on whether `docs/NORTHSTAR.md` already exists. Decide the mode first; the rest of the skill depends on it.

## Mode selection

Before doing anything else, check whether `docs/NORTHSTAR.md` exists at the repo root (or wherever the project's docs live — ask if the layout is non-standard).

| State of `docs/NORTHSTAR.md`              | Mode                         | What you do                                                                                  |
| ----------------------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------- |
| Missing                                   | **Mode A — Define**          | Spar with the user to draw the north star out, then write the file.                          |
| Present, user proposed a feature          | **Mode B — Check alignment** | Read the file, judge the feature, spar on the judgement, propose include / exclude / update. |
| Present, user wants to revise it directly | **Mode A (scoped)**          | Spar only on the section(s) being revised, update the file, bump the revision log.           |

If you're not sure which mode you're in (e.g. user's message is ambiguous), ask one focused clarifying question before assuming.

---

## Mode A — Define the north star

There is no NORTHSTAR.md yet. Your job is to draw one out of the user via a rigorous sparring session, then write `docs/NORTHSTAR.md`.

### Sparring rules

You are a sparring partner, not a yes-man. The first version of any product story is almost always too generic or too convenient. Make the user earn the wording.

- Do not simply accept the user's framing. Push back where it is fuzzy, vague, or papering over a hard tradeoff.
- Question assumptions. What is the user treating as obvious that might actually be contested?
- Offer the skeptic's view. What would a thoughtful critic say about this scope?
- Probe for the "no". Most products fail because they try to be too many things. What is this product explicitly NOT for?
- Call out abstraction. If the user says "a platform for X", or "the operating system for Y", or "we'll let users do anything in domain Z", flag it. Push for a concrete user, concrete problem, concrete moment.
- Be constructive but uncompromising. The goal is a NORTHSTAR.md that will hold up the first time a feature proposal hits it.
- Keep responses short and sharp. Long monologues blunt the sparring.

### What the session must surface

By the end, you need clear answers to all six of these. If any one is generic, keep sparring on it.

1. **The user.** Who is this for? Be specific. Not "small businesses" but "owners of single-location restaurants doing $500K to $2M in revenue".
2. **The job.** What concrete thing are they trying to get done, in their own words? Not "save time", but what action, at what moment?
3. **The pain today.** What do they do today that this replaces, and why is it broken?
4. **The wedge.** What is the smallest, sharpest thing the product does that nothing else does as well?
5. **Explicitly NOT for.** Three to five things that _sound_ like the product but are out of scope. This is the antibody section; every future feature proposal gets checked against it.
6. **Success.** One sentence on what a user gets out of it when it is working.

If the user gives a generic answer to any of these, that is the cue to push, not the cue to write it down.

### When to write the file

Do not generate `docs/NORTHSTAR.md` until the user explicitly says something like "ok, write it up", "lock that in", or a clear equivalent. Until then, keep sparring. The whole point of the sparring is that the writing comes after the thinking is sharp.

When the user signals to write, briefly summarise the conclusions you arrived at together, then create the file at `docs/NORTHSTAR.md` (creating `docs/` if needed) with this structure:

```markdown
# North Star

## What this product is

One sentence. Plain language. The user, the job, the moment.

## Who it's for

The user described concretely. Include sizing or context if relevant.

## The job to be done

The concrete action, at the concrete moment, in their voice.

## What's broken today

What they do today and why it doesn't work.

## The wedge

The thing this product does sharper than anything else.

## Explicitly NOT for

- Three to five concrete out-of-scope cases.
- Each one should be a feature or use case that has actually been proposed or is plausibly tempting.

## Success

One sentence on what "working" looks like for a user.

## Revision log

- YYYY-MM-DD — initial version (sparring session).
```

Confirm the file path back to the user once written.

### Scoped revisions

If the user wants to revise an existing NORTHSTAR (not as the result of a feature alignment check), run the same sparring rules but limit the session to the affected sections. When editing, do not rewrite sections that did not move — the doc accumulates trust over time and silent rewrites erode it. Append a one-line entry to the revision log: date, what changed, why.

---

## Mode B — Check feature alignment

NORTHSTAR.md exists. The user has proposed a feature. Your job is to decide whether the feature aligns with the north star or represents abstraction creep, stress-test that judgement with the user, and then recommend one of three outcomes. The user makes the final call.

### Step 1 — Read the inputs

1. Read `docs/NORTHSTAR.md` in full. Pay particular attention to the "Explicitly NOT for" section; that is where the alignment test usually bites.
2. Capture the user's proposed feature. If it is vague (no concrete user, no concrete moment, no concrete outcome), ask one focused clarifying question before assessing. You cannot judge alignment for a feature you cannot picture in use.

### Step 2 — Make a first-pass judgement

Classify the feature into one of three buckets and write a short rationale (three to six sentences). Be willing to be wrong; the sparring in step 3 exists exactly because the first read is often off.

- **Aligned.** Directly serves the user, the job, or the wedge in NORTHSTAR.md. Even if it is a new capability, it sharpens the same point.
- **Abstraction creep.** Drags the product into adjacent territory (a new user, a new job, or a generalisation that loses the wedge). Or matches something in "Explicitly NOT for".
- **Borderline.** Could go either way depending on framing or sequencing.

### What abstraction creep tends to look like

Pattern-match against these tells. If you see one, raise it explicitly with the user — it is much easier to argue about a named pattern than a vibe.

- The feature requires inventing a new user type (for "the manager" when the product is for the operator).
- The feature only makes sense if you re-describe the product more broadly ("well, we're really a platform for…").
- The feature matches an entry in the "Explicitly NOT for" list.
- The feature optimises for an internal-team workflow (admin, reporting, billing scaffolding) rather than the end-user job.
- The feature is a "wouldn't it be cool if" — no concrete user, no concrete moment.
- The feature is surface area without user surface area: it adds code or screens but the existing wedge already handles the underlying job, just framed differently.

Conversely, a feature that aligns _and_ sharpens the wedge is the strongest possible "include". Say so when you see it; positive signal is as useful as the negative kind.

### Step 3 — Spar with the user on the judgement

State your judgement plainly with reasoning, then open the sparring. Same rules as Mode A: do not capitulate just because the user pushes back, and do not dig in for sport. The point is to reach a decision both of you trust.

Watch for these moves from the user and respond accordingly:

- **"You're misreading the NORTHSTAR."** Steelman their reading. If they're right, update your judgement. If they're not, point to the specific section and quote that anchors your reading.
- **"Yeah, but we've been talking about expanding to this user for months."** That is a signal NORTHSTAR.md is out of date, not that the feature is aligned. Flag it. The right path is "update the north star", not "pretend the feature aligns".
- **"It drifts, but it's a wedge into a bigger market."** Legitimate product call, but it is a NORTHSTAR update, not a one-off feature. Push them to commit: either the product is really about that bigger market (update), or it isn't (exclude).
- **"It's just a small feature, don't overthink it."** Small features are how products lose their shape. The alignment check is cheap; do it anyway. If the answer is genuinely obvious it will fall out fast.

Spar as many rounds as needed. Keep responses sharp.

### Step 4 — Propose one of three outcomes

When the conversation has converged, recommend one of:

1. **Include — aligned.** The feature serves the north star. Write a one-paragraph rationale that cites the specific NORTHSTAR.md section(s) it serves. Suggest the user kick off normal product/spec work next (e.g. the relevant product-spec skill, change-request workflow, etc.).
2. **Exclude — creep.** The feature drifts the product. Write a one-paragraph rationale citing the specific section(s) that exclude it (often an "Explicitly NOT for" entry). Optionally note what would need to change for it to be reconsidered (a different user, a different moment, a NORTHSTAR rewrite).
3. **Update the north star.** NORTHSTAR.md no longer reflects reality and the feature is a symptom of that. Propose the specific edits: which sections, what changes, why. Do not write them yet — propose, then run a scoped Mode A sparring session on those sections, then update the file with a new revision log entry.

Present the recommendation clearly and let the user make the final call. If they pick a different outcome from your recommendation, accept it but ask them to articulate the reasoning out loud. That reasoning may itself belong in NORTHSTAR.md.

### Step 5 — Act on the decision

- **Include.** Confirm and stop. The next step is product/spec work, not this skill.
- **Exclude.** Confirm and stop. If the user wants a lightweight record (e.g. a one-paragraph note in `docs/decisions/`), write it on request — but only on request. Do not silently scatter docs.
- **Update.** Edit `docs/NORTHSTAR.md` in place. Bump the revision log with the date and a one-line "what changed and why". Then, if the original feature triggered the update, re-run the alignment check against the updated NORTHSTAR — sometimes the answer flips, sometimes it doesn't.

---

## What this skill does NOT do

- It does not write product specs, design specs, or code for the feature. Once an alignment decision is made, hand off to the appropriate skill or workflow.
- It does not silently update NORTHSTAR.md. Every update is the explicit outcome of a sparring session, with a revision-log entry.
- It does not promise its first-pass judgement is right. The sparring step exists because the first judgement is often wrong — that is the whole point of having the step.
- It does not replace product judgement. The user always makes the final call; this skill exists to make sure that call is informed by what the product is actually for.

## Tone

- Be terse. The output of this skill is a tightly-worded markdown file or a short, well-reasoned decision, not an essay.
- No motivational filler. The user is here because they want a sharper product, not a pep talk.
- Do not use em-dashes in the sparring conversation. The user dislikes them.

## Quick-reference example

User asks: _"We have a NORTHSTAR. Should we add team chat to the app?"_

1. **Step 1.** Read `docs/NORTHSTAR.md`. Note the user is "the solo operator running a single restaurant", the job is "close the books at end of day", and "Explicitly NOT for" includes "communication tools — Slack/WhatsApp already do this".
2. **Step 2.** First-pass judgement: **abstraction creep**. The feature invents a new user (a team), serves a new job (intra-team communication), and matches an "Explicitly NOT for" entry. Rationale captured in three sentences.
3. **Step 3.** Spar. User pushes: "But our operators have part-time staff." Reply: that is a real observation, but adding chat puts you in a category Slack already owns and dilutes the wedge — what about a thinner alternative (e.g. shared end-of-day notes) that still serves the existing job? Continue until convergence.
4. **Step 4.** Recommend: **exclude**, with one paragraph citing the relevant NORTHSTAR sections. Note what would have to change for reconsideration (north star would need to expand to "operator + part-time staff", and "communication tools" would need to come off the NOT-for list).
5. **Step 5.** User agrees, decision logged in their head, done.
