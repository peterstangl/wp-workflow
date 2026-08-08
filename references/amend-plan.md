> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

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

