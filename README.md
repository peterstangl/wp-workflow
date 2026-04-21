# wp-workflow

A Claude Code skill for running projects as a sequence of numbered work
packages (WPs), with a coarse project-level plan in `docs/PLAN.md` and a
fresh detailed plan per session.

## What it does

Three entry points, invoked via the `Skill` tool with `skill: "wp-workflow"`:

- **`bootstrap`** — interview-driven setup of a new repo. Produces
  `CLAUDE.md` (standing conventions, resource map, repo layout, WP status)
  and `docs/PLAN.md` (scope, per-WP done criteria).
- **`implement <WP-id>`** — per-session routine for one WP. Enters plan
  mode first, reads the repo's `CLAUDE.md` + `docs/PLAN.md`, writes a
  detailed per-WP plan for approval, implements after approval, then
  updates `CLAUDE.md` + `docs/PLAN.md` before handing back for the user
  to commit.
- **`archive-plan`** — compresses a long `docs/PLAN.md` by copying it
  verbatim to a dated archive and leaving only a summary + link for
  closed WPs.
- **`retrospect`** — harvests lessons from a project that has used this
  skill and folds them back into `SKILL.md` or the templates.

See [SKILL.md](SKILL.md) for the full specification.

## Install

Clone the repo directly into your user-level skills directory:

```sh
git clone <remote-url> ~/.claude/skills/wp-workflow
```

Claude Code picks the skill up automatically. No other setup needed.

To update later: `git -C ~/.claude/skills/wp-workflow pull`.

## Invariants

Baked into the skill, applied on every entry point:

- **The user owns version control.** The skill never runs mutating git
  commands and never commits for you.
- **Edits stay inside the current repo.** The only exception is
  `retrospect`, which edits this skill's own files — that is what
  invoking it authorises.

## Layout

```
wp-workflow/
├── README.md                  ← this file
├── SKILL.md                   ← skill spec (Claude Code reads this)
└── templates/
    ├── CLAUDE.md.tmpl         ← scaffold for a project's CLAUDE.md
    └── PLAN.md.tmpl           ← scaffold for a project's docs/PLAN.md
```

## Future: plugin packaging

This directory is currently shipped as a bare skill. If it ever grows to
include related agents, hooks, or slash commands worth bundling, it can
be migrated into a Claude Code plugin: add a `.claude-plugin/plugin.json`
manifest at the repo root and move `SKILL.md` + `templates/` into
`skills/wp-workflow/`. Until then, the flat layout keeps the install path
(clone into `~/.claude/skills/wp-workflow/`) as simple as possible.
