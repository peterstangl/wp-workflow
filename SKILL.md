---
name: wp-workflow
description: Bootstrap and run a work-package-based project workflow. Use when the user wants to set up a new repo with a CLAUDE.md + docs/PLAN.md structure, work on a specific WP ("implement WP3", "work on WP2", "continue with WP5", "start WP7"), add or re-scope a WP in the plan, or compress a long docs/PLAN.md once many WPs are complete. Trigger whenever the user mentions work packages, WPs, or asks to bootstrap / implement / amend / archive project plans in this style, even if they don't name the skill. Also invoke with `sign-off`, but only when the user says the literal words "sign off" or "sign-off": that entry point verifies the session is really finished and refuses if it is not, so status questions like "are you done" must NOT be routed to it. The value of invoking is getting the session into plan mode *before* any code or plan changes, which is the central guardrail of the workflow.
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

**4. Only an `implement` session implements.** A work package's deliverables are produced, and its box checked `[x]`, only inside an `implement <WP-id>` session that has passed its own per-WP `ExitPlanMode` and run that WP's verification. `bootstrap` and `amend-plan` author plans, never code — every WP they write ships `[ ]`, however small it looks and even if this session believes it could satisfy it now. **That naming is an example, not the scope**: the first sentence says *only* inside an `implement` session, so it binds `retrospect` and `archive-plan` equally, and a reader who takes the second sentence as the enumeration will conclude the unnamed entry points are unconstrained. One session read it that way and asked, which
is the only reason it surfaced; had it concluded instead of asking, nothing would have. Note that `retrospect` *does* write files — this skill's own, under `~/.claude/skills/`, authorised by the exception in invariant 2 — which is what makes the confusion available: the test is never whether a session writes, it is whether what it writes is a work package's deliverable. This rule lives here, not in the generated `CLAUDE.md`: the "one session per WP" line there cannot bind the bootstrap session that is only now writing it. It does not re-gate a session's own closing bookkeeping — `implement` Step 5 refining upcoming WPs and calling `archive-plan` in-line are that session's already-approved work, not a second entry point's.

## Picking the entry point

Use `args` to choose:

- `bootstrap` — new repo, first-time setup.
- `implement <WP-id>` — the user says "implement WP3", "work on WP2", "continue with WP5", or similar. Aliases: `run-wp`, `wp`.
- `amend-plan` — add, re-scope, re-order, or drop a WP in `docs/PLAN.md`, outside an implement session.
- `archive-plan` — compress a long `docs/PLAN.md`.
- `retrospect` — harvest lessons from a project that has used this skill and fold them back into `SKILL.md` or the templates.
- `sign-off` — end the session deliberately. **Triggered only by the literal phrase "sign off" or "sign-off"**, never by a question such as "are you done" and never by your own judgement that the work looks finished.

If the user's phrasing matches one of these but they didn't name the skill, invoke anyway — getting into plan mode before code *or plan* changes is what the skill buys.

**Each entry point's procedure lives in its own file, read only when that entry point is used.** The order is always the same: **register in the coordination log and read the coordination reference, then call `EnterPlanMode`** (four of the five do), then read the procedure inside plan mode.

| entry point | enters plan mode | read at registration | procedure |
|---|---|---|---|
| `bootstrap` | yes | [references/parallel-sessions.md](references/parallel-sessions.md) | [references/bootstrap.md](references/bootstrap.md) |
| `implement <WP-id>` | yes | [references/parallel-sessions.md](references/parallel-sessions.md) | [references/implement.md](references/implement.md) |
| `amend-plan` | yes | [references/parallel-sessions.md](references/parallel-sessions.md) | [references/amend-plan.md](references/amend-plan.md) |
| `archive-plan` | no | [references/parallel-sessions.md](references/parallel-sessions.md) | [references/archive-plan.md](references/archive-plan.md) |
| `retrospect` | yes | [references/parallel-sessions.md](references/parallel-sessions.md) | [references/retrospect.md](references/retrospect.md) |
| `sign-off` | no | already read; this is a departure, not an arrival | [references/sign-off.md](references/sign-off.md) |

The third column repeats deliberately. A session that read only its own row still sees it, and the table is
the part of this file that is demonstrably acted on: asked afterwards, a session that had missed the
coordination reference entirely reported reading `SKILL.md`, "which the Skill tool loads", and its own
procedure file, "because the router named it for my entry point". Prose telling a reader to go and read
something else is what that session skipped, so the pointer belongs in the structure that worked.

The invariants above and the coordination rules below apply to every one of them, so they stay here rather than in any single file.
## Before any entry point: the coordination log

The user may be running several sessions on one repo at once, each on a different WP, with no channel
between them. Coordination is an append-only log at **`docs/coordination/`**, inheriting whatever
storage mode `docs/` already has.

**Register on arrival, unconditionally, and before `EnterPlanMode`.** Append one line naming your work
package, creating the directory if it is absent. This is the one write that precedes plan mode: it
touches no project file, and deferring it until after plan mode would leave a planning session
invisible for as long as planning takes, which here has run to hours.

**Do these five things as you register.** They are here rather than in the reference because this file is
loaded for you and the reference is read only if you choose to read it, and because every one of them,
omitted, is paid by a *peer* rather than by you. Each was measured on one repository in one day.

1. **Arm a watch on the log, replaying from the beginning rather than from the current end**, and re-arm
   it whenever control returns from the user. Arming at the end silently declares everything above it
   read. A rule you must remember to run is not a channel: one session took four mid-turn messages
   without reading and ran forty minutes blind, missing a claim takeover inside its own files.
2. **Take every timestamp from `date`; never compose one.** The two sessions running without a watch
   wrote every one of the four blocks dated ahead of the clock, because nothing was showing them the
   real time.
3. **Keep your read offset in a file**, never in context, since context is exactly what a long gap
   destroys.
4. **Put the owner in each `CLAIM:` and `RELEASE:` line** (`CLAIM: <paths> | <you>`) and spell the paths
   identically in both. A release that does not match its claim string never clears it, and a claim with
   no owner cannot be told from a quotation of someone else's.
5. **Append with `>>` and nothing else.** Never read the log, transform it and write it back: any peer's
   append landing between your read and your write is erased, with no gap, no shrinkage and no alert.

Invisible planners are the case that matters. A session that is implementing reads the log, sees
nobody, concludes it is alone and skips its **invalidation notice** — while a planner is quietly
measuring numbers that are about to move. That is the failure the notice exists to prevent, and
registering only before your first *write* would not prevent it, because planning does not write.

Do not first try to decide whether anyone else is running: you cannot, which is rule 2 below, and if
every session concludes it is probably alone then no session ever registers, no log is ever created,
and coordination never begins in a project that has not used it before. That deadlock is silent.
Registering costs one line, and it is what makes rule 2 answerable at all.

Three rules govern reading the log, and with the five actions above they are all a session needs unless
two are implementing at once:

1. **Read it at every point where the user has acted.** The principle, not the list: any moment the
   user speaks or answers is a moment you may have been idle and the world may have moved, and it is the
   only signal you get, since you cannot perceive elapsed time. That includes a fresh prompt, an
   `AskUserQuestion` answer, an `ExitPlanMode` approval *or rejection*, **and a message that arrives
   mid-turn while you are working** — which is the one a closed list invites you to skip. One session
   took four mid-turn messages without treating any as a return point and ran forty minutes blind,
   missing a claim takeover inside its own files. If in doubt whether something counts, it counts; the
   read is one command.
2. **You cannot know you are alone without looking**, so that read comes before any conclusion about
   who else is running.
3. **Announce before writing anything a peer might also be editing**, and check the log's claim lines
   first. Claims are advisory, not locks.

Everything else — the message vocabulary, the watch that turns the log into a push channel, commit
choreography, and a list of designs that were tried and discarded — is in
[references/parallel-sessions.md](references/parallel-sessions.md). **Read it in the same step as
registering, unconditionally, and do not try to decide first whether you need it.**

The reason is the one that makes registration unconditional, one level up. Deciding whether you need it
means deciding whether anyone else is running, which is rule 2's unanswerable question, and the log, the
watch and the timestamp discipline that would answer it are all specified in that file. So a conditional
read is a read that happens after the damage. Measured across five sessions on one repository, a
conditional gate opened for nobody on the skill's own account: two sessions read the file in full only
after a human told them a peer existed, one never opened it and inferred its contents from the sentence
pointing at it. Be exact about what that cost, because most of the same day's defects had other causes:
the two sessions that never read it ran with **no watch at all**, and produced every block that was dated
ahead of the clock. Both costs landed on peers rather than on them, which is the asymmetry that makes the
read unconditional. The sessions that *had* read it still wrote broken watch filters, and every session
including this file's author filed claim lines nobody could attribute to an owner. **So reading is
necessary and is not sufficient**, and this rule does not claim otherwise: a caution interrupts an error,
only a mechanism prevents it.

At roughly 11k tokens it is much the largest read in this skill. **That is the file's problem to fix by
being shorter, never this rule's to fix by being conditional.**

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
