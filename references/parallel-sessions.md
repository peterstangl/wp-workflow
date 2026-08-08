# Working alongside other sessions

Read this when more than one session is actually running on the repo. A solo session needs only the
three rules in `SKILL.md`; nothing here applies to it.

Everything below came out of two sessions running concurrently on one project for four hours. Almost
none of it is invention. Every surviving **mechanism** was proposed by the user rather than designed by
either session; the sessions invented two **message shapes** that survived, `INVALIDATES` and the
two-list commit handover, and seven session-authored designs that did not. The final section lists
those, because both sessions independently re-derived and re-retracted the same ideas having been told
nothing about them.

## Where the log lives

`docs/coordination/`, inheriting whatever storage mode `docs/` already has. That gets it right in all
three modes the skill supports, with no new question: tracked, excluded via `.git/info/exclude`, or
symlinked to a private context repo. When `docs/` is tracked in the project repo, exclude the
subfolder — a log inside a shared history is not disposable, and the duty to move conclusions out of it
quietly lapses.

It has to be retrofittable, since most projects were bootstrapped long before any of this existed.
Every project the skill has touched already has `docs/`, so this is one `mkdir` by whichever session
arrives first — **not** by whichever session "needs it", which is a condition no session can evaluate
and which deadlocks the whole mechanism at startup. In a project with no log, absence is ambiguous
between "nobody coordinates here" and "nobody has registered yet", and a session that resolves that
ambiguity toward *alone* prevents the next session from resolving it any other way. Hence
unconditional registration on arrival: it converts an unanswerable question into a one-line append, and
it is the only thing that gets a project from never-coordinated to coordinated.

What a session actually does, first time, in an existing project:

```
docs/coordination/            <- mkdir, plus one exclude line if docs/ is tracked
docs/coordination/log.md      <- append: date, work package, what you are about to touch
```

The second session to arrive finds that line, and coordination has begun without either session
having had to guess.

## How much coordination, and when

Plan mode is **enforced by the harness**, not promised by a session. That single fact does most of the
work here, because it means "I am planning" is a claim another session can rely on without trusting
anyone:

| who is running | what is needed |
|---|---|
| any number **planning** | registration only — none of them can write, so no claims are needed |
| one **implementing**, N **planning** | registration, plus invalidation notices from the implementer |
| two or more **implementing** | the full claim protocol below |

Registration is in every row, including the first, and that is not a formality. It is what makes the
second row work: an implementer decides whether to post an invalidation notice by reading the log, so a
planner that has not registered is a planner the implementer will not warn. Since planning writes
nothing, a session that registers only before its first write stays invisible for the whole of the
period in which it is most exposed — it is measuring numbers, and those are exactly what an
invalidation notice is about. Hence registration precedes `EnterPlanMode` rather than the first write.

The third row is the expensive one, and it is worth avoiding rather than managing: see *Commit
choreography*.

## The log itself

One append-only file. Append with `cat >>`, never `Write` — the whole design depends on nobody
truncating it, and that is the one discipline that has never failed in practice. Never rewrite or edit
another session's block.

**Generate every timestamp with `TS=$(date '+%Y-%m-%d %H:%M')` inside the appending command.** Do not
type one. Both sessions in the founding exchange typed timestamps by hand and both drifted *forwards*,
one by half an hour, which put a block in the log dated in the future and sent the user hunting for a
message that looked missing. One of them had actually run `date`, seen `02:21`, and typed `02:24`
anyway. The general form is worth more than the rule: **anything with no source in your context must
come from a tool call, and when a tool disagrees with your intuition the tool is not a suggestion.**

Order the log by **append position**, never by the timestamps in the headers. Append order is also what
arbitrates two sessions racing for the same claim, which is the one property that makes a single shared
file better than a file per session.

### Reading it

Two different reads, for two different needs:

- **The lock set** — grep or `awk` the *whole* log and replay `CLAIM:`/`RELEASE:` with last-one-wins.
  Output is a handful of paths whatever the log's size, so it costs nothing and cannot miss a block you
  skipped.
- **The prose** — read incrementally from an offset stored in a **file** in your scratchpad. Context is
  exactly what a long gap destroys, so an offset held in context is worthless after one.

Incremental reading is only sound *because* the log is append-only. That is the real reason for the
never-truncate rule.

### A watch, so the log pushes

A `Monitor` on the log turns a dead drop into a channel, and without one the user has to relay
messages by hand. Emit **one event per new block header or claim line, never the prose** — piping the
diff puts every block into context twice, and the prose is then read on demand from the offset.

Include a branch that reports the file **shrinking or disappearing**. A growth-only watch is silent
exactly when someone has destroyed another session's history, and silence is indistinguishable from
quiet. This is the only automatic check on the append-only rule.

Arm it as soon as a peer may exist; take it down only when the peer has **ended**, not when it declares
itself finished. In the founding exchange both sessions posted "closing" and then exchanged five more
substantive blocks.

**So do not sign off with a declaration; sign off with a criterion.** "Nothing further" was said four
times and was wrong every time. A criterion is checkable, survives being wrong, and tells the reader
what your silence will mean.

**When a review exchange keeps producing the same class of defect, stop checking the last edit and
sweep the artefact for the class.** Checking edit *n+1* against edit *n* cannot terminate, because
every fix is prose and prose carries new instances; a sweep can, and it says what is left. In the
founding exchange that converted an open-ended correspondence into a bounded audit in one step: six
uniqueness claims found, four sound, two fixed, class closed.

Two things about that rule, in the spirit of the rest of this file. **The trigger is declared, not
measured** — the class had recurred *three* times before anyone swept, so there is no observed point at
which switching pays, and "after the second instance" is a choice a reader may move. And a sweep is
complete only **for its own grep**: a different defect class needs a different sweep.

**Draw the criterion on relevance, and mark the evidentiary class rather than gating on it:** *"I will
post again if I find something that bears on the artefact, saying whether I can demonstrate it or only
observed it."* The first version of this convention gated on demonstrability — "if I find a defect I
can demonstrate" — and that is a filter, not a criterion. It silently discards the most interesting
category there is, something you noticed that bears on the work and cannot prove, while *looking*
disciplined, which makes it worse than a loose declaration. Its author committed exactly that
suppression within one turn of writing it, on its first use, and the user had to ask why.

**A watch dies silently** — when its process exits, when the session is resumed later, when the harness
tears it down. Nothing announces this, and the symptom is indistinguishable from a quiet peer. Knowing
that is not enough to prevent it: the author of this file knew its watch was dead, said it would
re-arm, and did not, then had to be told by the user that replies were going unread.

So do not make re-arming a thing to remember. **Bind it to the trigger you already have: whenever
control returns from the user, re-read the log *and* re-arm the watch if it is not running.** Same
moment, same rule, no new discipline.

**Test whether it is running like this, and not the obvious way:**

```bash
pgrep -fa '<a marker string from the watch loop>' | grep -v pgrep
```

The naive `pgrep -fc '<marker>'` is **always at least 1**, because the shell running the check has the
pattern in its own command line and matches itself. Measured: the naive form returned `3` where the
truth was `2`, the third being the checking process. A session using it concludes its watch is alive
unconditionally, the re-arm never fires, and the rule silently does nothing while appearing to work,
which is worse than having no rule because it manufactures confidence. The harness may *also* report a
dead background task when a session resumes; that notice is easy to miss, so treat this test as the
reliable one.

The general form is worth keeping: **a rule containing a predicate the session cannot evaluate is not a
rule.** If you write "if X", say how to test X, **and then run that test in the state where it must
fail.** The last clause is the one that executes; "satisfy yourself that it can return false" asks for
reasoning about a predicate's range, and nobody does that, which is exactly how the self-match survived
review here.

For this case the failing state is a pattern matching no process at all:

```bash
pgrep -fc 'zzz-no-such-watcher-zzz'                    # -> 1   the naive form
pgrep -fa 'zzz-no-such-watcher-zzz' | grep -vc pgrep   # -> 0   the working form
```

The naive predicate returns 1 for a pattern that matches **nothing**, so it is provably incapable of
reporting "not running" — shown in one command, at the moment the check is written, with no reasoning
required.

That also puts the watch in its proper place. **Read-on-return is mandatory; the watch is
best-effort.** A dead watch then costs only mid-turn timeliness, never correctness, because the next
return from the user picks up everything missed. Treat any design that depends on the watch being alive
as broken.

## Message types

Split by lifecycle, because that is what decides whether a message needs revoking:

**Standing** — someone may act on it for a period, so it must be revoked: presence (`ARRIVED`), claims,
intent or plan, an invalidation notice, and "I am waiting for X". **A stale standing statement is worse
than none, because it causes action.** A forgotten "waiting for" makes another session pause at every
checkpoint indefinitely.

**Events** — no revocation: a checkpoint reached, a commit landed, a correction, a finding.

Most revocations should be *derived* rather than remembered: departing revokes everything that session
held. Rules that depend on remembering get broken, usually within minutes of being written down.

**A counterweight, because nearly every fix in this file replaces a stated rule with an executable one,
and that could be read as "stated cautions are worthless".** They are not, and the distinction is
sharper than "usually". Naming a failure mode does not stop you *committing* it — one session
diagnosed its own grep-the-phrase-not-the-concept error and repeated it twice more. But on the third
repetition the stated caution stopped it *publishing* the false claim: it hesitated, re-checked, and
found the phrase was there and split across a line break. **Publication is the harmful half**, since a
claim that reaches another party has left the loop where either could catch it. So prefer a mechanism
where one exists, and keep the stated caution for where none does — it works exactly at the moment you
are about to assert something.

Two that are easy to miss. **A plan or intent that the user has since changed is a standing statement
gone stale** — when a scope decision arrives, the sessions that were told the old scope still believe
it. And a user decision that constrains anything beyond the current WP should be written into the repo
*when it is made*, not at WP close, because until then it exists only in one session's transcript.

### The one message type worth naming

**`INVALIDATES` — "a number you are about to quote is about to change."** It is the highest-value thing
one session can tell another, and it is one of the two message shapes the founding exchange invented
that survived, the other being the two-list commit handover below. It is also the one a commit barrier does *not* make unnecessary, because planning stays
concurrent: a plan measured against a tree that is about to move is the failure that survives every
other precaution.

It was named in that exchange, explicitly ranked first, and then dropped for two hours while both
sessions built locks instead. Name it, then build for it.

### Claims are advisory

A claim says someone intends to edit something. It does not stop you.

**There is no reliable way to tell whether a session is still alive.** Directory mtimes are not a
liveness signal — a live session's own directory can read days stale, because a directory's mtime
tracks entries added or removed in it, not writes inside its subdirectories. So do not invent a
staleness threshold; a declared cut wearing a measured cut's clothes is worse than none.

Gate on **reversibility** instead. For work that `git` makes visible and revertible: announce the
takeover in the log and proceed. For anything not cheaply reversible: **ask the user**, who is the only
party who knows which sessions they started.

Takeover is two-sided. The taker announces; the holder finds out by reading the log when control
returns from the user, which is the same rule as everything else and needs no special resume branch.

## Commit choreography

The user owns `git`, so a session can only ever say it is ready. Both halves are mechanisable: a
`READY-TO-COMMIT` line arrives as an event on the peer's watch, and `git rev-parse HEAD` costs about
two milliseconds, so a background `until` loop gives one notification when the commit lands.

Serialising implementation behind commits makes several problems structurally impossible rather than
merely managed: two sessions re-pinning the same constants, two builds fighting over one document's
`.aux` files, a data regeneration running under another session's read, and a commit that mixes two
work packages so neither can be reviewed.

Two limits, both learned the hard way:

- **It does not fix plan shelf life.** Planning stays concurrent by design, so a plan measured against
  a moving tree is untouched by it. That is what `INVALIDATES` is for.
- **Write sets are discovered, not declared.** A plan cannot predict which files an implementation will
  touch; the founding exchange produced three surprise files in one WP. So any scheme gated on
  declaring your files in advance leaks, which argues for the coarse form ("implementation is
  serialised") over a finer one.

And it converts a race into a stall on a human, which can be overnight. That is usually the right trade,
since a stalled session is safe and a raced one is not, but say it out loud, and pair it with the
mitigation the barrier already permits: the waiting session plans, reads and reviews.

### Checkpoints in the checklist

Put **explicit commit checkpoints as their own items** in the WP's todo list. Do not require every item
boundary to be commit-coherent: coherent units genuinely span several items, and forcing the
decomposition to serve the commit rhythm makes the plan worse. One extraction needed two items to land
together or the document would not build; one reframing spanned four, and committing after any one left
the documents contradicting each other.

Make the checkpoint **conditional**: pause and commit *if another session has asked for it*, otherwise
carry on. Costs nothing when running alone. The request is a standing statement posted whenever, read
at the checkpoint — no synchronous handshake.

**Whole-WP commits stay the default for a single session.** Smaller commits are for coordination, not a
general improvement.

### Two lists at handover

When handing the tree over, post **two** lists, not one: the files you changed **on purpose**, and the
files you only **re-pinned or regenerated as a consequence**. One session's second list was five test
files whose constants moved because the observable set moved, plus every generated table and figure.
To a reviewer those look like unrelated churn; saying so up front is the difference between a
reviewable diff and a suspicious one.

This is the mitigation for the failure the section above names. A commit that mixes two work packages
is hard to review, and when one did land that way, the two-list handover was the only thing that made
it reviewable at all. Post it even when the commit is clean, because the second list is exactly what a
reviewer cannot infer.

## Tried and discarded

With reasons, because two sessions independently re-derived and re-retracted most of this having been
told nothing about it. The reasons matter more than the items.

- **A file per session, and a file per message.** Both make truncation structurally impossible, and
  truncation never once occurred. Per-session files also discard the total order that arbitrates
  competing claims, and both trade the discipline that held perfectly (never-truncate) for the one that
  failed repeatedly (timestamps). The deciding argument is not that it never happened, which is the
  smoke-alarm fallacy: it is that the log being disposable already caps the damage.
- **Gating parallel implementation on disjoint write sets.** Write sets are discovered, not declared,
  so the gate cannot be evaluated when the decision must be made.
- **A staleness threshold, and a resume/gap-detection branch.** A session cannot perceive a gap, and
  there is no measurable staleness criterion. Both dissolve into "read when control returns".
- **Heartbeats and liveness detection.** Measured as unreliable; see *Claims are advisory*.
- **"Push makes staleness worse."** A watch strictly adds information. What degrades is a *behaviour* —
  treating the absence of a `RELEASE` event as evidence of a live claim. The fix is one inference rule,
  not machinery.
- **"A second session is for verification."** Cross-session checking is a byproduct of shared files
  forcing communication, and is largely redundant with an adversarial subagent, which a single session
  already has. Two sessions are for implementing different work packages.

One caution that outranks the protocol. Both checking mechanisms available to a session — a subagent,
and a peer session — are **frame-internal by construction**: a subagent inherits your framing through
the prompt you write it, and a peer inherits it through a long shared conversation. A framing error is
invisible from inside the frame, because the frame is what decides which artefacts count as checkable.
In the founding exchange every deep error was caught by the user and none by either mechanism —
**five consecutive times**, including the last one, where a session's own sign-off criterion excluded
the very finding it had just made and neither session noticed, though one of them was holding both
halves. To check a frame you need a reader that was not told the frame.

**A check's characteristic failure is to confirm the hypothesis it was testing.** A broken check
reports the defect it was looking for, and that reads as a finding rather than as a fault. Verifying
one refactor here, an off-by-one in the extraction reported 15.5 kB of content lost, and a pattern
that failed to match reported all five files as differing; both were the instrument, and either
published would have been a serious false accusation against a correct change. What caught it was not
inspecting the check but noticing its output **contradicted an independent measurement**: bodies down
15.5 kB cannot coexist with a total up 1.7 kB. So **suspect a failing check before its subject, and
confirm against a quantity measured a different way.** Note the direction: the two rules below are
about *false negatives*, something present reported missing; this is the mirror, a *false positive*
for a defect, manufactured by the instrument.

**When a claim pairs a measured quantity with a declared threshold, the measurement attracts the
checking and the threshold rides in free.** Both sessions here declined to ship a finding because a
file was "at 295 of its 300-line limit". The 295 was genuinely measured with `wc -l` and reported as
verified; the 300 had been deleted from the rule some hours earlier and nobody looked. The measurement
is checkable, so checking it feels like diligence and yields a real number, while the threshold is the
half that actually decides — and the failure modes are not comparable: a bad measurement is wrong by
units, a threshold that does not exist is wrong by everything. Check the *deciding* half, which is
usually the one presented as settled.

Note what this adds to the measured-versus-declared discipline rather than restating it: labelling a
number "declared" marks it as a choice, and therefore as something not to re-derive, which is precisely
the property that let a deleted threshold survive. **The label predicts where verification will not
go.** Both instances of this, and the sign-off filter that quietly gated on demonstrability, shared one
disguise worth naming: **the failure arrived dressed as discipline.** Restraint, a budget, a criterion
— each looked like rigour and each was the error.

**And a stopping condition for "cheapest sufficient", which that rule otherwise lacks.** Each failure
of a cheap instrument looks like a small fix, so sessions keep patching it. The signal is not the
number of failures, which would be another declared threshold: it is whether **consecutive failures
have unrelated causes.** One bug twice means fix the bug. Two *different* bugs mean the instrument is a
poor fit for the question rather than buggy, and it is time to switch, which is available at the second
failure without counting to anything. In the case above the two faults were an `awk` boundary error and
a pattern-anchor mismatch, unrelated, and the question in fact wanted a language with real string
handling rather than nested shell quoting.

One practical corollary, since checking an artefact is what this whole document rests on: **grep for
the concept, not for a contiguous phrase.** A claim you are verifying may be present but wrapped across
a line break, so an exact-phrase search returns nothing and you conclude a fix is missing that is
sitting there. That error occurred three times here, twice published. Search on a distinctive fragment,
or on two keywords, and read the surrounding lines before concluding absence.
