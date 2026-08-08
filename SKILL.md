---
name: wp-workflow
description: Bootstrap and run a work-package-based project workflow. Use when the user wants to set up a new repo with a CLAUDE.md + docs/PLAN.md structure, work on a specific WP ("implement WP3", "work on WP2", "continue with WP5", "start WP7"), add or re-scope a WP in the plan, or compress a long docs/PLAN.md once many WPs are complete. Trigger whenever the user mentions work packages, WPs, or asks to bootstrap / implement / amend / archive project plans in this style, even if they don't name the skill. The value of invoking is getting the session into plan mode *before* any code or plan changes, which is the central guardrail of the workflow.
---

# wp-workflow

A meta-workflow skill for projects structured around numbered work packages (WPs). It encodes four routines:

1. **Bootstrap** a new repo in plan mode: produce a repo-local `CLAUDE.md` (standing conventions, resource map, repo layout, WP status) plus `docs/PLAN.md` (scope, per-WP done-criteria).
2. **Implement one WP at a time**: every "implement WPx" session re-enters plan mode first, writes a detailed per-WP plan, gets approval, implements, then updates `CLAUDE.md` and `docs/PLAN.md` so the user can commit.
3. **Amend** the plan in plan mode: add, re-scope, re-order, or drop a WP outside an implement session, numbering it so its id says whether it refines an existing WP or is independent of it.
4. **Archive** `docs/PLAN.md` when it grows long: copy verbatim to a dated archive file and replace the live file with a compressed summary plus archive link.

The structural idea is **nested plans**. A coarse, stable plan lives in `docs/PLAN.md`; a fresh, detailed plan per session lives in the plan file that plan mode provides. That split keeps each session focused, keeps the main plan readable, and records why decisions were made.

## Invariants that apply to every entry point

These four rules apply everywhere in this skill and override anything below that might read otherwise.

**1. The user owns version control.** Never run `git add`, `git commit`, `git push`, `git rm`, `git reset`, `git stash`, or any other git command that mutates history or the working tree, and never propose to. Summarise what changed in plain text and let the user commit. Read-only git commands (`git status`, `git diff`, `git log`) are fine to confirm state.

**2. Stay inside the current repository.** Only edit files inside the repo the user is working in. Do not write to, delete from, or reorganise anything outside it — including the user's home dir, sibling repos, or this skill's own files — without an explicit instruction in the current session. One standing exception: a session may keep its own bookkeeping (such as the coordination log's read offset, see [references/parallel-sessions.md](references/parallel-sessions.md)) in its own scratchpad directory. The coordination log itself lives *inside* the repo and needs no exception.

**3. Context is a budget, not a free resource.** Prefer narrow reads (`offset`/`limit`, targeted `Grep`) over whole-file reads. For anything large or broad, delegate to `Agent(subagent_type="Explore")` with a specific question — the subagent's summary enters context, not the raw file. At WP close, check file sizes and invoke `archive-plan` if `CLAUDE.md > 500 lines` or `docs/PLAN.md > 800 lines`.

**Reach for the cheapest instrument that resolves the question, and refine only if it does not.** It cuts three ways, and the founding episode committed all three in one chain:

- **Do not ask a peer what you can check yourself.** One session asked another "did your WP touch a pinned constant?" when `git status` crossed with its own record of what it had edited answered it locally in seconds. That was the first link, and without it neither of the others happens.
- **Do not convene a fleet for what one command answers.** The asked session routed that question through five agents, when `git show --stat HEAD -- 'numerics/test_*.py'` settles it in two seconds. Delegation is for breadth, not for thoroughness theatre.
- **Do not hedge where you could have a fact.** Rather than wait, the asking session told the user "I don't think it did" — a guess offered where two seconds of checking was available.

**4. Only an `implement` session implements.** A work package's deliverables are produced, and its box checked `[x]`, only inside an `implement <WP-id>` session that has passed its own per-WP `ExitPlanMode` and run that WP's verification. `bootstrap` and `amend-plan` author plans, never code — every WP they write ships `[ ]`, however small it looks and even if this session believes it could satisfy it now. This rule lives here, not in the generated `CLAUDE.md`: the "one session per WP" line there cannot bind the bootstrap session that is only now writing it. It does not re-gate a session's own closing bookkeeping — `implement` Step 5 refining upcoming WPs and calling `archive-plan` in-line are that session's already-approved work, not a second entry point's.

## Picking the entry point

Use `args` to choose:

- `bootstrap` — new repo, first-time setup.
- `implement <WP-id>` — the user says "implement WP3", "work on WP2", "continue with WP5", or similar. Aliases: `run-wp`, `wp`.
- `amend-plan` — add, re-scope, re-order, or drop a WP in `docs/PLAN.md`, outside an implement session.
- `archive-plan` — compress a long `docs/PLAN.md`.
- `retrospect` — harvest lessons from a project that has used this skill and fold them back into `SKILL.md` or the templates.

If the user's phrasing matches one of these but they didn't name the skill, invoke anyway — getting into plan mode before code *or plan* changes is what the skill buys.

**Each entry point's procedure lives in its own file, read only when that entry point is used.** The order is always the same: **register in the coordination log, then call `EnterPlanMode`** (four of the five do), then read the procedure inside plan mode.

| entry point | enters plan mode | procedure |
|---|---|---|
| `bootstrap` | yes | [references/bootstrap.md](references/bootstrap.md) |
| `implement <WP-id>` | yes | [references/implement.md](references/implement.md) |
| `amend-plan` | yes | [references/amend-plan.md](references/amend-plan.md) |
| `archive-plan` | no | [references/archive-plan.md](references/archive-plan.md) |
| `retrospect` | yes | [references/retrospect.md](references/retrospect.md) |

The invariants above and the coordination rules below apply to every one of them, so they stay here rather than in any single file.
## Before any entry point: the coordination log

The user may be running several sessions on one repo at once, each on a different WP, with no channel
between them. Coordination is an append-only log at **`docs/coordination/`**, inheriting whatever
storage mode `docs/` already has.

**Register on arrival, unconditionally, and before `EnterPlanMode`.** Append one line naming your work
package, creating the directory if it is absent. This is the one write that precedes plan mode: it
touches no project file, and deferring it until after plan mode would leave a planning session
invisible for as long as planning takes, which here has run to hours.

Invisible planners are the case that matters. A session that is implementing reads the log, sees
nobody, concludes it is alone and skips its **invalidation notice** — while a planner is quietly
measuring numbers that are about to move. That is the failure the notice exists to prevent, and
registering only before your first *write* would not prevent it, because planning does not write.

Do not first try to decide whether anyone else is running: you cannot, which is rule 2 below, and if
every session concludes it is probably alone then no session ever registers, no log is ever created,
and coordination never begins in a project that has not used it before. That deadlock is silent.
Registering costs one line, and it is what makes rule 2 answerable at all.

Three rules, and they are all a session needs unless two are implementing at once:

1. **Read it whenever control returns from you to the user** — a prompt, an `AskUserQuestion` answer,
   an `ExitPlanMode` approval *or rejection*. Those are the only moments you can have sat idle, so they
   are the only moments the world can have moved without you noticing.
2. **You cannot know you are alone without looking**, so that read comes before any conclusion about
   who else is running.
3. **Announce before writing anything a peer might also be editing**, and check the log's claim lines
   first. Claims are advisory, not locks.

Everything else — the message vocabulary, the watch that turns the log into a push channel, commit
choreography, and a list of designs that were tried and discarded — is in
[references/parallel-sessions.md](references/parallel-sessions.md). Read it when more than one session
is actually running; skip it otherwise.

---

## Improving this skill outside `retrospect`

When the user says something like *"update wp-workflow so that X"* or *"add X to the skill based on what we just learned"*, treat it the same way as `retrospect`: enter plan mode, propose the specific edit with a one-line rationale, get approval, then apply it to `~/.claude/skills/wp-workflow/`.

**"The same way" includes two things from `retrospect` that are spelled out here rather than left to inherit**, because a rule whose applicability depends on how a reader interprets "the same way" is the same defect class as a predicate with no stated test: it looks like a mechanism and is a wish.

- **Claim `SKILL.md` in the coordination log before editing, and release it after.** These files sit outside every repo, so nothing else mediates two sessions editing them at once.
- **The size check**: `SKILL.md` under 5k tokens (`wc -c`, divide by four), `references/` unbounded, and report where the mass sits. Move material out rather than compress it.

Both matter at least as much here as in `retrospect`, since most skill edits arrive mid-session as *"update wp-workflow so that X"* rather than as a formal `retrospect` — an observation from one session's history, not a measured distribution.

Two further defaults that matter:

- **Process lessons go into `SKILL.md`**; **structural lessons go into a template file**. A lesson about how a session should flow (e.g. "always check for an existing similar function before proposing a new module") edits `SKILL.md`. A lesson about what every project's scaffold should contain (e.g. "add a *Testing conventions* section") edits a template.
- **Prefer one load-bearing sentence over accumulation.** If a new rule duplicates or softens an existing one, rewrite the existing one instead of adding alongside. The templates in particular get used by every future bootstrap — keeping them lean keeps generated `CLAUDE.md` files readable.

---

## Why plan-mode-first for "implement WPx"

Two reinforcements make this guard hard to bypass:

1. This skill's `implement` entry starts with `EnterPlanMode`.
2. The generated `CLAUDE.md` tells every future session: *"For any prompt of the form 'implement WPx', 'work on WPx', or 'continue with WPx', invoke the `wp-workflow` skill with `implement <WPx>`. Do not begin implementation before `ExitPlanMode` approval on a per-WP plan."*

Either path lands in plan mode, so free-text prompts in a fresh session still go through the right door.
