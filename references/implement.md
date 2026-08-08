> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

## Entry point: `implement <WP-id>`

Goal: complete one work package cleanly, leaving the repo in a state where the user can review and commit.

### Step 1 — enter plan mode immediately

Call `EnterPlanMode` **before any reads of implementation files**. This is the skill's central guard. The project-level `docs/PLAN.md` says *what* a WP is; the per-session plan file says *how* to do it in this repo right now. Skipping plan mode collapses those two levels and the session drifts into implementation before the user has seen the approach.

### Step 2 — read project context, scoped to the WP

Every line you pull into context shrinks the budget for actual implementation later — the constraint is *resources*, not the project docs.

In order:

1. The repo's `CLAUDE.md` — read in full.
2. `docs/PLAN.md` — read in full if it's short enough to be cheap; focus your attention on the requested WP and its direct dependencies but don't be afraid to skim neighbours for context.
3. The resources the WP entry references — **be selective**:
   - Not every WP needs every resource the project lists in its *Resource map*. Open only what this specific WP's goal requires.
   - Small files (< ~2000 lines): read directly, including orientation skims — scanning a short reference to get your bearings is cheap and often the right move.
   - Large files (multi-MB data files, long papers, legacy modules where you only need a narrow fact): don't orient in-session. Delegate to `Agent(subagent_type="Explore")` with the specific question you're trying to answer — the subagent does the orientation + drill-down and returns only a distilled summary. If you don't yet know what you're looking for, the subagent's first job can be to locate the relevant section (abstract, TOC, grep result) and report back.
   - Before opening a large file directly, pause and ask: could a subagent answer this?

You are collecting just enough to write a defensible per-WP plan, not trying to understand the whole repo.

### Step 3 — write the per-WP plan

Into the plan file, write:

- **Context** — why this WP exists, what "done" means for it. Summarise from `docs/PLAN.md`; don't restate the full project.
- **Approach** — concrete steps. Reference existing functions/modules to reuse with paths; name the files you'll create or modify.
- **Verification** — the exact commands or checks that prove the WP's done-criteria are satisfied.

Use `AskUserQuestion` for real ambiguities. Don't pad with clarifying questions to fill the phase.

### Step 4 — `ExitPlanMode` and implement

After approval, execute the plan. Keep all edits inside the current repo.

**Numbers measured while planning have a shelf life.** Approval can arrive minutes or days after the
plan was written, and another session may have committed in between. Any figure the plan quotes from
the repo — a count, a rank, a reach, a test constant — is an *input with an expiry*, not a fact.
Re-measure anything you are about to act on or write into a document. One plan carried a headline that
had moved from 16 to 0 by the time it ran, and it was caught only because that plan happened to say
"re-measure in Phase 2" rather than carrying the number forward.

### Step 5 — close the WP (no commit)

As the final step of the session, before handing back for the user to commit:

1. Update `CLAUDE.md`: mark the WP `[x]` with a one-line summary and links to the closing code/tests. Add any new modules, data files, or conventions the WP introduced. **Keep completed-WP entries to one line each** — the git history and the closing diff are the record of how the WP was done; `CLAUDE.md` is a living guide, not a changelog.
2. Update `docs/PLAN.md`: refine upcoming WPs if what you learned changes them. Don't retell the WP — the diff is the record.
3. **Check the size of both files.** Run `wc -l CLAUDE.md docs/PLAN.md`. If either exceeds the archive threshold (see below), run the `archive-plan` entry point in-line before handing off. Do this as the final sub-step so the user reviews one coherent diff: WP changes + archive, in that order.
4. **Move any durable conclusion out of the coordination log**, into `CLAUDE.md`, `docs/PLAN.md` or a design document, and leave a pointer behind. Then release every claim the session still holds. The log is disposable by design, so anything left in it is lost — and a finding that only exists there is unrecoverable from the conclusions that cite it, because who established what, and against which evidence, is not reconstructable after the fact.

Then summarise in chat what changed and what the user should check before committing. Do not run any git mutation.

**Archive thresholds.** Trigger `archive-plan` when either `CLAUDE.md > 500 lines` or `docs/PLAN.md > 800 lines`. These are rules of thumb, not hard limits — use judgement if the file is genuinely all still load-bearing.

