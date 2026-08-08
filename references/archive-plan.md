> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

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

