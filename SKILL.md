---
name: wp-workflow
description: Bootstrap and run a work-package-based project workflow. Use when the user wants to set up a new repo with a CLAUDE.md + docs/PLAN.md structure, work on a specific WP ("implement WP3", "work on WP2", "continue with WP5", "start WP7"), or compress a long docs/PLAN.md once many WPs are complete. Trigger whenever the user mentions work packages, WPs, or asks to bootstrap / implement / archive project plans in this style, even if they don't name the skill. The value of invoking is getting the session into plan mode *before* any code changes, which is the central guardrail of the workflow.
---

# wp-workflow

A meta-workflow skill for projects structured around numbered work packages (WPs). It encodes three routines:

1. **Bootstrap** a new repo in plan mode: produce a repo-local `CLAUDE.md` (standing conventions, resource map, repo layout, WP status) plus `docs/PLAN.md` (scope, per-WP done-criteria).
2. **Implement one WP at a time**: every "implement WPx" session re-enters plan mode first, writes a detailed per-WP plan, gets approval, implements, then updates `CLAUDE.md` and `docs/PLAN.md` so the user can commit.
3. **Archive** `docs/PLAN.md` when it grows long: copy verbatim to a dated archive file and replace the live file with a compressed summary plus archive link.

The structural idea is **nested plans**. A coarse, stable plan lives in `docs/PLAN.md`; a fresh, detailed plan per session lives in the plan file that plan mode provides. That split keeps each session focused, keeps the main plan readable, and records why decisions were made.

## Invariants that apply to every entry point

These two rules apply everywhere in this skill and override anything below that might read otherwise.

**1. The user owns version control.** Never run `git add`, `git commit`, `git push`, `git rm`, `git reset`, `git stash`, or any other git command that mutates history or the working tree, and never propose to. Summarise what changed in plain text and let the user commit. Read-only git commands (`git status`, `git diff`, `git log`) are fine to confirm state.

**2. Stay inside the current repository.** Only edit files inside the repo the user is working in. Do not write to, delete from, or reorganise anything outside it — including the user's home dir, sibling repos, or this skill's own files — without an explicit instruction in the current session.

**3. Context is a budget, not a free resource.** Prefer narrow reads (`offset`/`limit`, targeted `Grep`) over whole-file reads. For anything large or broad, delegate to `Agent(subagent_type="Explore")` with a specific question — the subagent's summary enters context, not the raw file. At WP close, check file sizes and invoke `archive-plan` if `CLAUDE.md > 500 lines` or `docs/PLAN.md > 800 lines`.

## Picking the entry point

Use `args` to choose:

- `bootstrap` — new repo, first-time setup.
- `implement <WP-id>` — the user says "implement WP3", "work on WP2", "continue with WP5", or similar. Aliases: `run-wp`, `wp`.
- `archive-plan` — compress a long `docs/PLAN.md`.
- `retrospect` — harvest lessons from a project that has used this skill and fold them back into `SKILL.md` or the templates.

If the user's phrasing matches one of these but they didn't name the skill, invoke anyway — getting into plan mode before code changes is what the skill buys.

---

## Entry point: `bootstrap`

Goal: produce an initial `CLAUDE.md` and `docs/PLAN.md` tailored to the user's project.

### Step 1 — enter plan mode

Call `EnterPlanMode` before any reads of the target repo. The drafts go into the plan file, not the repo. Files land only after `ExitPlanMode` approval.

### Step 2 — interview the user

Ask only what you actually need; don't run down a checklist mechanically. What usually matters:

- What is the project and why does it exist? (one paragraph of "core idea")
- What inputs/resources does the project build on? (external docs, data files, reference implementations)
- What is in scope and out of scope?
- Are there hard rules you want every session to follow? (data provenance, tooling purity, naming, coding style)
- What work packages do you see, at a first pass? (titles + one-line done-criteria is enough)
- Does this project use Python, LaTeX, both, or neither? (controls the starter `.gitignore` — see step 4)

Use `AskUserQuestion` for multiple-choice clarifications; plain text questions otherwise.

### Step 3 — draft the artefacts

Read [templates/CLAUDE.md.tmpl](templates/CLAUDE.md.tmpl) and [templates/PLAN.md.tmpl](templates/PLAN.md.tmpl) and fill them in. Put both drafts *into the plan file* so the user reviews them in one place. Do not write into the target repo yet.

**CLAUDE.md must cover:**

1. A "Read this first, then `docs/PLAN.md`" pointer.
2. A one-paragraph *Core idea* — enough for a fresh agent to understand why the code is shaped the way it is.
3. *Standing conventions* — numbered list. Common entries, copy only what applies: data-provenance rule ("script-driven data"), tooling/language purity constraints, naming conventions, and the *living docs* rule: "every WP ends with the executing agent updating `CLAUDE.md` and, if appropriate, `docs/PLAN.md`, as part of the WP's changes, before the user commits". The living-docs rule is load-bearing — never drop it.
4. *Resource map* — one short paragraph per external reference, each ending with a "read when…" clause so future agents can decide whether to open the file.
5. *Repository layout* tree.
6. *Execution model* block with the plan-mode-first rule and the "user owns git" rule — the exact wording matters; copy from the template.
7. *Work-package status* checklist: `[ ]` per planned WP with a one-line summary.

**docs/PLAN.md must cover:**

- Scope (in / out).
- A pointer back to `CLAUDE.md` for conventions (don't duplicate them).
- Per-WP entries: goal, inputs, deliverables, **done criteria**, cross-references.
- A visible slot at the top for the archive-link line (left empty until `archive-plan` runs).

Templates are scaffolds, not boilerplate to inject verbatim. Prefer a shorter, truthful CLAUDE.md to a long template-shaped one — delete placeholders that don't apply.

### Step 4 — `ExitPlanMode`

After approval, write the artefacts into the target repo and stop. Do not commit.

Artefacts to write:

1. `CLAUDE.md` (from the draft).
2. `docs/PLAN.md` (from the draft).
3. A starter `.gitignore` at the repo root, **only if the project uses Python and/or LaTeX** and the repo does not already have a `.gitignore`. Source the content from:
   - [templates/gitignore/python.gitignore](templates/gitignore/python.gitignore)
   - [templates/gitignore/latex.gitignore](templates/gitignore/latex.gitignore)

   If both apply, concatenate them with a clear section separator (e.g. `# --- Python ---` and `# --- LaTeX ---` headers) so the origin of each block stays obvious when the user edits it later. If neither applies, skip the file. If a `.gitignore` already exists, leave it alone and mention in the summary that it was preserved — the user can merge by hand.

---

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

### Step 5 — close the WP (no commit)

As the final step of the session, before handing back for the user to commit:

1. Update `CLAUDE.md`: mark the WP `[x]` with a one-line summary and links to the closing code/tests. Add any new modules, data files, or conventions the WP introduced. **Keep completed-WP entries to one line each** — the git history and the closing diff are the record of how the WP was done; `CLAUDE.md` is a living guide, not a changelog.
2. Update `docs/PLAN.md`: refine upcoming WPs if what you learned changes them. Don't retell the WP — the diff is the record.
3. **Check the size of both files.** Run `wc -l CLAUDE.md docs/PLAN.md`. If either exceeds the archive threshold (see below), run the `archive-plan` entry point in-line before handing off. Do this as the final sub-step so the user reviews one coherent diff: WP changes + archive, in that order.

Then summarise in chat what changed and what the user should check before committing. Do not run any git mutation.

**Archive thresholds.** Trigger `archive-plan` when either `CLAUDE.md > 500 lines` or `docs/PLAN.md > 800 lines`. These are rules of thumb, not hard limits — use judgement if the file is genuinely all still load-bearing.

---

## Entry point: `archive-plan`

Goal: compress `CLAUDE.md` and/or `docs/PLAN.md` without losing history, so every future session starts with a smaller auto-loaded footprint.

The two files serve different roles and compress differently:

- `docs/PLAN.md` — per-WP detail for closed work goes away; open WPs stay verbatim.
- `CLAUDE.md` — paragraph-long completed-WP entries collapse to one line each; standing conventions, resource map, and repo layout stay verbatim.

Never delete; always back up first.

### Step 1 — require a clean tree

Check `git status`. Refuse to proceed on a dirty working tree: the archive operation must land as one reviewable diff (or be stacked on top of the same WP's changes, when invoked from `implement` Step 5). Ask the user to commit or stash first. Do not stash for them.

### Step 2 — copy both files verbatim to dated backups

For each file that crosses its threshold, or that the user explicitly names, copy it to a dated archive:

- `docs/PLAN.md` → `docs/PLAN-archive-YYYY-MM-DD.md`
- `CLAUDE.md` → `docs/CLAUDE-archive-YYYY-MM-DD.md` (goes into `docs/` so the repo root stays clean)

No edits to archives — they are the backup. If a same-day archive already exists, append a short suffix (`-2`, `-3`, …); do **not** overwrite.

### Step 3 — rewrite the live files

**`docs/PLAN.md`** becomes:

- Archive-link line at the top: *Full history: [PLAN-archive-YYYY-MM-DD.md](PLAN-archive-YYYY-MM-DD.md).*
- The scope section and the pointer to `CLAUDE.md` conventions.
- A *Completed WPs* section with one bullet per `[x]` WP — one line of summary each.
- All still-open WPs in full, unchanged.

**`CLAUDE.md`** becomes:

- Archive-link line near the top (above *Work-package status*): *Full completed-WP history: [docs/CLAUDE-archive-YYYY-MM-DD.md](docs/CLAUDE-archive-YYYY-MM-DD.md).*
- Every section *except* *Work-package status* stays verbatim.
- In *Work-package status*, every completed WP collapses to a single line: `- [x] **WPx** — <one-line summary>`. Links to closing code/tests are fine if they fit on that line; paragraph-long recaps of what was done go to the archive only.
- Open WPs stay as they were.

Keep the two files non-redundant: if a piece of information is in `CLAUDE.md`, don't duplicate it in `docs/PLAN.md`, and vice versa. The two summaries for a closed WP — the `CLAUDE.md` line and the `docs/PLAN.md` bullet — should say different things (CLAUDE.md: "what lives in the repo now", PLAN.md: "what was delivered"); if they end up the same, drop the PLAN.md one.

### Step 4 — report and stop

Report the line-count reduction for each file and ask the user to eyeball both new files plus the archives before committing. Do not commit for them.

---

## Entry point: `retrospect`

Goal: turn lessons from a project that has used this skill into concrete improvements to `SKILL.md` or the templates, so the next project starts ahead of where this one started.

### Step 1 — enter plan mode

Call `EnterPlanMode` before any edits to the skill. All proposed changes go into the plan file first.

### Step 2 — gather signal

Read, in this order:

1. The current repo's `CLAUDE.md` — look at the standing-conventions list and the resource map. Conventions added mid-project are often the best candidates for promoting into the generic template.
2. The closed WPs in `CLAUDE.md` — the one-line summaries often reveal patterns (e.g. "every WP needed a cross-check oracle" suggests adding a convention slot for that).
3. Recent per-WP plan files under `~/.claude/plans/` for this repo — read the ones from the last few WPs, not everything. Look for steps that felt like they should've been automated, moments where the user corrected the approach, or verification recipes that recurred.
4. Feedback memories under the current session's memory dir (`feedback_*.md`) — if any rule there is generic enough to apply to every wp-workflow user, it belongs in the skill.

### Step 3 — propose edits

Into the plan file, write a short list of proposed changes. For each:

- **What:** the exact edit — a patch-sized change to `SKILL.md` or a template.
- **Why:** the observation from step 2 that motivates it, with a pointer (a filename or WP id) so the user can verify the evidence.
- **Where it belongs:** `SKILL.md` for *process* lessons (how a session should flow, what to ask, what to avoid), template files for *structural* lessons (what every project's `CLAUDE.md` or `docs/PLAN.md` should contain).

Bias toward **fewer, load-bearing additions** over a long list. A skill that accumulates every lesson becomes unreadable fast. If a candidate lesson is borderline, leave it out — the user can always invoke `retrospect` again later when the pattern has recurred in a second project.

### Step 4 — `ExitPlanMode` and apply

After the user approves, edit `SKILL.md` and/or the templates under `~/.claude/skills/wp-workflow/`. This is the one place in this skill where editing outside the current repo is expected — the user's invocation of `retrospect` is the explicit per-session instruction that authorises it.

Summarise what changed, and suggest the user version-control their `~/.claude/skills/wp-workflow/` directory if they aren't already.

---

## Improving this skill outside `retrospect`

When the user says something like *"update wp-workflow so that X"* or *"add X to the skill based on what we just learned"*, treat it the same way as `retrospect`: enter plan mode, propose the specific edit with a one-line rationale, get approval, then apply it to `~/.claude/skills/wp-workflow/`.

Two defaults that matter:

- **Process lessons go into `SKILL.md`**; **structural lessons go into a template file**. A lesson about how a session should flow (e.g. "always check for an existing similar function before proposing a new module") edits `SKILL.md`. A lesson about what every project's scaffold should contain (e.g. "add a *Testing conventions* section") edits a template.
- **Prefer one load-bearing sentence over accumulation.** If a new rule duplicates or softens an existing one, rewrite the existing one instead of adding alongside. The templates in particular get used by every future bootstrap — keeping them lean keeps generated `CLAUDE.md` files readable.

---

## Why plan-mode-first for "implement WPx"

Two reinforcements make this guard hard to bypass:

1. This skill's `implement` entry starts with `EnterPlanMode`.
2. The generated `CLAUDE.md` tells every future session: *"For any prompt of the form 'implement WPx', 'work on WPx', or 'continue with WPx', invoke the `wp-workflow` skill with `implement <WPx>`. Do not begin implementation before `ExitPlanMode` approval on a per-WP plan."*

Either path lands in plan mode, so free-text prompts in a fresh session still go through the right door.
