> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

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

