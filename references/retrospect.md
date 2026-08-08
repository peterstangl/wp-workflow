> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

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

**Triage on mechanism, not on regret.** A lesson earns a rule when it names something structural that a session could not have avoided by trying harder: a predicate nobody can evaluate, a check that cannot return false, a claim composed before its evidence existed. Everything else was carelessness, and **a skill that absorbs carelessness as procedure gets longer without getting better** — a reader pays for every line, and "be careful with numbers" costs them exactly what a rule that would have caught something costs. When a candidate reduces to "and then I should have looked", leave it out.

**A lesson that passes that test but has been seen in only one project goes to [candidate-rules.md](candidate-rules.md), and nowhere else.** Apply the triage to what *leaves* the skill as well as to what enters it: "defer it" is not a decision unless the deferral names a place that survives and a condition someone can evaluate. Three candidates were once deferred into a project's coordination log — excluded from version control, disposable, and unreadable by the second project whose arrival was supposed to promote them. That is an unevaluable predicate wearing a decision's clothes. The candidates file is outside every repository, survives every project, and is read by this entry point, which is the one that would promote its contents.

**The skill has a size budget too, it is documented rather than invented, and it is checked on both of the paths that grow it** — here, and in *Improving this skill outside `retrospect`* below.

Anthropic's Agent Skills guidance budgets by **tokens and load frequency**, not by lines: the `description` is always in context (~100 tokens), `SKILL.md` loads in full on **every invocation** and should stay **under 5k tokens**, and files under `references/` load only when read and have *no practical limit*. Check with `wc -c` and divide by four for a rough token count; a line count is a bad proxy, because dense prose at ~90 characters per line costs more than twice as much per line as prose at ~40.

Two consequences that are easy to get backwards. **`SKILL.md` is the file to protect**, since every entry point's text is loaded even when a session uses only one of them — a `bootstrap` section costs every `implement` session that never runs it. And **`references/` is the destination, not a thing to cap**: capping it defeats the progressive disclosure the architecture is built on. So when `SKILL.md` is over budget, move material *out* rather than compress it.

Report where the mass sits, not just the total: `awk '/^## /{...}'` over the headings shows which section to move first. A skill that tells every project to watch its file sizes while exempting its own would be holding a standard it does not apply.

### Step 4 — `ExitPlanMode` and apply

After the user approves, edit `SKILL.md` and/or the templates under `~/.claude/skills/wp-workflow/`. This is the one place in this skill where editing outside the current repo is expected — the user's invocation of `retrospect` is the explicit per-session instruction that authorises it.

Nothing mediates these edits: the files are outside every repo, so no coordination log covers them and two sessions editing the skill would collide silently. Claim `SKILL.md` in the coordination log before editing and release it after. That is a convention rather than a mechanism, and it is the weakest guard in this skill; say so if it matters. The same applies to the other path that edits these files, *Improving this skill outside `retrospect`* below — which is the more travelled one, so the guard matters there at least as much.

Summarise what changed, and suggest the user version-control their `~/.claude/skills/wp-workflow/` directory if they aren't already.

