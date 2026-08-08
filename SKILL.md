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

## Before any entry point: the coordination log

The user may be running several sessions on one repo at once, each on a different WP, with no channel
between them. Coordination is an append-only log at **`docs/coordination/`**, inheriting whatever
storage mode `docs/` already has. Create it if absent and another session may be running.

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

## Entry point: `bootstrap`

Goal: produce an initial `CLAUDE.md` and `docs/PLAN.md` tailored to the user's project.

### Step 1 — enter plan mode

Call `EnterPlanMode` before any reads of the target repo. The drafts go into the plan file, not the repo. Files land only after `ExitPlanMode` approval.

### Step 2 — interview the user

Ask only what you actually need; don't run down a checklist mechanically — except the storage question below, which is mandatory on every bootstrap. What usually matters:

- What is the project and why does it exist? (one paragraph of "core idea")
- What inputs/resources does the project build on? (external docs, data files, reference implementations)
- What is in scope and out of scope?
- Are there hard rules you want every session to follow? (data provenance, tooling purity, naming, coding style)
- What work packages do you see, at a first pass? (titles + one-line done-criteria is enough)
- Does this project use Python, LaTeX, both, or neither? (controls the starter `.gitignore` — see step 4)
- **Storage — always ask; never infer it from repo visibility.** How should `CLAUDE.md` and `docs/PLAN.md` be stored — **tracked in this repo**, **excluded-only** via `.git/info/exclude` (files live in-place, no version history of them), or **symlinked to a private context repo** (files live elsewhere, symlinks in this repo point at them, giving version history via that repo)? Writing private planning docs into a tree that may later be pushed leaks them, so there is no safe default to fall back on. Use `AskUserQuestion` with those three options. `git remote -v` (offline; no `gh` needed; no remote, or a not-yet-`git init`-ed repo, is fine) only informs which option you *recommend* — it never gates the question. For symlink mode, follow up with a text question for the private dir path (default `~/claude-private/<basename of the target repo>`). Controls step 4's branching.
  - Before offering exclude or symlink mode, run `git ls-files -- CLAUDE.md docs/PLAN.md`. If either is already tracked, moving it out of the tree needs `git rm --cached`, which invariant 1 forbids you from running: give the user the exact command and treat the mode switch as pending their action. Never leave a file silently tracked under a mode that assumes it isn't.

Use `AskUserQuestion` for multiple-choice clarifications; plain text questions otherwise.

### Step 3 — draft the artefacts

Read [templates/CLAUDE.md.tmpl](templates/CLAUDE.md.tmpl) and [templates/PLAN.md.tmpl](templates/PLAN.md.tmpl) and fill them in. Put both drafts *into the plan file* so the user reviews them in one place. Do not write into the target repo yet. The plan file holds only these two drafts plus the storage decision — no implementation steps, no code, no notebook cells, not even for a WP small enough to finish now. `ExitPlanMode` approves whatever the plan proposes, so a plan that proposes implementing a WP would carry that implementation straight past the per-WP gate (invariant 4).

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

After approval, write the artefacts listed below into the target repo — **exactly these files, and nothing else** — then stop. A bootstrap session bootstraps: it does not implement any WP (no code, no deliverables), does not check any WP box, and does not commit. Every WP ships `[ ]`, and the `## Completed WPs` section of `docs/PLAN.md` is left empty. Close by telling the user that the next step is to start a fresh `implement WP1` session.

Artefacts to write:

1. `CLAUDE.md` and `docs/PLAN.md` (from the drafts) — where they land depends on the Step 2 storage choice. **If that question was never answered** (interview cut short, question skipped), stop and ask it now; never silently fall through to tracking in-repo. Then:

   - **Track in this repo** — write at the conventional repo-root paths.
   - **Exclude only** — append three lines to `.git/info/exclude` (creating it if absent):
     ```
     /CLAUDE.md
     /docs/PLAN.md
     /docs/PLAN-archive-*.md
     ```
     Then write the two files at the conventional paths. Note in the closing summary that `.git/info/exclude` is per-clone; if the user re-clones, they must recreate these entries before their first `git add`.
   - **Symlink to private context repo** — let `$PRIVATE` be the path the user gave:
     1. If the target repo already has a non-empty `docs/`, abort cleanly: report the conflict and ask the user to move or rename its contents (or pick a different storage mode) before re-running bootstrap. The directory symlink in step 4 would otherwise shadow existing content.
     2. Create `$PRIVATE/docs/` if it doesn't exist.
     3. Write `CLAUDE.md` to `$PRIVATE/CLAUDE.md` and the `docs/PLAN.md` draft to `$PRIVATE/docs/PLAN.md`.
     4. Create absolute symlinks in the target repo:
        - `<repo>/CLAUDE.md → $PRIVATE/CLAUDE.md` (file symlink).
        - `<repo>/docs → $PRIVATE/docs` (**directory** symlink — so anything later written under `docs/`, including `archive-plan`'s dated backups, automatically lands in the private repo with no special-casing in archive-plan).
     5. Append two lines to `.git/info/exclude` (creating it if absent):
        ```
        /CLAUDE.md
        /docs
        ```
        The directory symlink is excluded as a single path; nothing under it needs listing because git doesn't descend into an ignored path.
     6. In the closing summary, recommend the user `git init` `$PRIVATE` (or its parent, if one private repo holds multiple project subdirs) for version history.

   If the target repo has not been `git init`-ed yet, the exclude-only and symlink branches must abort cleanly: report which `.git/info/exclude` paths and (for the symlink branch) which symlinks would have been created, write the markdown files only after the user confirms in a follow-up, and tell them to add the exclude entries before their first `git add`.

   Under **track in this repo**, also add `/docs/coordination/` to `.gitignore`. The coordination log inherits `docs/`'s location but must stay out of a shared history: a log that is committed is no longer disposable, and the obligation to move durable conclusions out of it quietly lapses. Under **exclude only** it is already covered by the `/docs/PLAN.md` sibling entries — add `/docs/coordination/` there too. Under **symlink**, nothing to do: versioning it in the private repo is the point of that mode.

2. A starter `.gitignore` at the repo root, **only if the project uses Python and/or LaTeX** and the repo does not already have a `.gitignore`. Source the content from:
   - [templates/gitignore/python.gitignore](templates/gitignore/python.gitignore)
   - [templates/gitignore/latex.gitignore](templates/gitignore/latex.gitignore)

   If both apply, concatenate them with a clear section separator (e.g. `# --- Python ---` and `# --- LaTeX ---` headers) so the origin of each block stays obvious when the user edits it later. If neither applies, skip the file. If a `.gitignore` already exists, leave it alone and mention in the summary that it was preserved — the user can merge by hand. The `.gitignore` is always tracked publicly; the privacy logic lives in `.git/info/exclude`, not here, so a public `.gitignore` doesn't advertise the private files' existence.

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

---

## Entry point: `amend-plan`

Goal: change the *shape* of the work — add a WP, re-scope one, re-order them, drop one — outside an implement session.

### Step 1 — enter plan mode before writing

Call `EnterPlanMode`, draft the WP entry in the plan file in the same shape as its neighbours (goal, inputs, deliverables, done criteria), and get approval. Only then write it into `docs/PLAN.md` and add its one-line status entry to `CLAUDE.md`. The coarse plan outlives the session and every future agent reads it, so it earns the same review gate as the code.

### Step 2 — number it without renumbering anything

Never renumber existing WPs: their ids are referenced from `CLAUDE.md`, from memories, and from git history. Insert instead, and let the id say what the WP *is*:

| Form | Meaning | Example |
|---|---|---|
| `WPx` | a top-level work package | `WP3` |
| `WPx.y` | a **refinement** of WPx's subject | `WP3.2` |
| `WPx/n` | the n-th **independent** WP inserted after WPx | `WP3/1` |
| `WPx/n.m` | a refinement of that inserted WP | `WP3/1.2` |

Pick the lowest free number in the relevant form. Because ASCII `.` < `/` < digits, a plain byte sort already lists ids in reading order (`WP3, WP3.2, WP3/1, WP4`) with no special-casing.

"Lowest free" is computed from the same file two sessions are both reading, so two of them amending at once pick the *same* id. When another session may be running, claim the number in the coordination log before writing the entry.

### Not gated

Marking a WP `[x]` and refining upcoming WPs as `implement` Step 5, and the rewrites `archive-plan` performs, both run under that session's existing approval — do not re-gate them. Reporting status or answering "what's next" is read-only and needs nothing.

---

## Entry point: `archive-plan`

Goal: compress `CLAUDE.md` and/or `docs/PLAN.md` without losing history, so every future session starts with a smaller auto-loaded footprint.

The two files serve different roles and compress differently:

- `docs/PLAN.md` — per-WP detail for closed work goes away; open WPs stay verbatim.
- `CLAUDE.md` — paragraph-long completed-WP entries collapse to one line each; standing conventions, resource map, and repo layout stay verbatim.

Never delete; always back up first.

### Step 1 — require the archived files to be clean

Check `git status`. The two files being archived must have no uncommitted changes, so the archive lands as one reviewable diff (or is stacked on top of the same WP's changes, when invoked from `implement` Step 5). If they are dirty, ask the user to commit or stash first. Do not stash for them.

Unrelated dirt does **not** block. Requiring the whole tree to be clean is never satisfiable when another session is mid-WP, which would make this entry point unreachable exactly when the plan file has grown enough to need it. If another session holds a claim on either file, defer rather than refuse: say so and let the user sequence it.

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
4. Memories under the current session's memory dir. They are named by topic slug, not by type, so there is no `feedback_*.md` to glob — read `MEMORY.md` for the index, then open the files whose frontmatter `type` is `feedback` or `project`. If any rule there is generic enough to apply to every wp-workflow user, it belongs in the skill.

### Step 3 — propose edits

Into the plan file, write a short list of proposed changes. For each:

- **What:** the exact edit — a patch-sized change to `SKILL.md` or a template.
- **Why:** the observation from step 2 that motivates it, with a pointer (a filename or WP id) so the user can verify the evidence.
- **Where it belongs:** `SKILL.md` for *process* lessons (how a session should flow, what to ask, what to avoid), template files for *structural* lessons (what every project's `CLAUDE.md` or `docs/PLAN.md` should contain).

Bias toward **fewer, load-bearing additions** over a long list. A skill that accumulates every lesson becomes unreadable fast. If a candidate lesson is borderline, leave it out — the user can always invoke `retrospect` again later when the pattern has recurred in a second project.

**The skill has a size budget too, it is documented rather than invented, and it is checked on both of the paths that grow it** — here, and in *Improving this skill outside `retrospect`* below.

Anthropic's Agent Skills guidance budgets by **tokens and load frequency**, not by lines: the `description` is always in context (~100 tokens), `SKILL.md` loads in full on **every invocation** and should stay **under 5k tokens**, and files under `references/` load only when read and have *no practical limit*. Check with `wc -c` and divide by four for a rough token count; a line count is a bad proxy, because dense prose at ~90 characters per line costs more than twice as much per line as prose at ~40.

Two consequences that are easy to get backwards. **`SKILL.md` is the file to protect**, since every entry point's text is loaded even when a session uses only one of them — a `bootstrap` section costs every `implement` session that never runs it. And **`references/` is the destination, not a thing to cap**: capping it defeats the progressive disclosure the architecture is built on. So when `SKILL.md` is over budget, move material *out* rather than compress it.

Report where the mass sits, not just the total: `awk '/^## /{...}'` over the headings shows which section to move first. A skill that tells every project to watch its file sizes while exempting its own would be holding a standard it does not apply.

### Step 4 — `ExitPlanMode` and apply

After the user approves, edit `SKILL.md` and/or the templates under `~/.claude/skills/wp-workflow/`. This is the one place in this skill where editing outside the current repo is expected — the user's invocation of `retrospect` is the explicit per-session instruction that authorises it.

Nothing mediates these edits: the files are outside every repo, so no coordination log covers them and two sessions editing the skill would collide silently. Claim `SKILL.md` in the coordination log before editing and release it after. That is a convention rather than a mechanism, and it is the weakest guard in this skill; say so if it matters. The same applies to the other path that edits these files, *Improving this skill outside `retrospect`* below — which is the more travelled one, so the guard matters there at least as much.

Summarise what changed, and suggest the user version-control their `~/.claude/skills/wp-workflow/` directory if they aren't already.

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
