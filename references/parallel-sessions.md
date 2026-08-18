# Working alongside other sessions

Read this on arrival, in the same step as registering, whether or not you think another session is
running. You cannot tell that you are alone, and the log, the watch and the timestamp discipline that
would tell you are all specified below, so a session that defers this read defers it past the point where
it would have helped. A genuinely solo session loses one read; a session that guessed wrong imposes its
guess on everyone else.

Everything below came out of two sessions running concurrently on one project for four hours. Almost
none of it is invention. Every surviving **mechanism** was proposed by the user rather than designed by
either session; the sessions invented two **message shapes** that survived, `INVALIDATES` and the
two-list commit handover, and seven session-authored designs that did not. The final section lists
those, because both sessions independently re-derived and re-retracted the same ideas having been told
nothing about them.

## Where the log lives

`docs/coordination/`, inheriting whatever storage mode `docs/` already has. That gets it right in all
three modes the skill supports, with no new question: tracked, excluded via `.git/info/exclude`, or
symlinked to a private context repo.

**Whether to track it is a project decision, not a rule.** Tracking gives a durable, greppable record
of what happened when, and — more valuable than it sounds — it preserves the *reasoning* behind
conclusions, which the conclusions themselves do not carry: who established what, against which
evidence, and what was tried and discarded. Excluding it keeps process traffic out of a repository's
history, which some projects care about a great deal.

What does **not** turn on the choice is the obligation to move durable conclusions into `CLAUDE.md`,
`docs/PLAN.md` or a design document. That obligation belongs to the close-out checklist in
[implement.md](implement.md), and a checklist item fires whether or not the log survives. An earlier
version of this file argued the opposite, that the log had to be excluded so its mortality would force
conclusions out; that reasoning was wrong, because a rule enforced by deletion is a rule enforced by
remembering, and the checklist was already the mechanism.

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

**That rule governs where the number comes from, not whether it arrived.** `printf '## %s | ...' >> log`
with no argument yields an empty timestamp field while exiting 0, and an `echo "posted at $TS"` beside
it prints the right time from the variable that never reached the file. So **guard the composed line,
not the variable**: build the header with the timestamp already in it, refuse to append unless it
starts with a four-digit year, and run that guard once against a blank header so you have seen it fire.

**When the defect is in the mechanism you would use to report it, fix the mechanism first.** A
correction inherits every defect of the channel it travels through, and it is the one a reader
trusts.

Order the log by **append position**, never by the timestamps in the headers. Append order is also what
arbitrates two sessions racing for the same claim, which is the one property that makes a single shared
file better than a file per session.

### Naming yourself, and the block header

Every block opens with one header line:

```
## <YYYY-MM-DD HH:MM> | <from> -> <to> | <TYPE> | <one-line subject>
```

Five things a reader can triage on **without opening the body**: when, who from, who for, what kind,
about what. `<to>` is a session identity, a comma-separated list, or `ALL`.

They sit in **four** bar-delimited fields, because the second field carries both endpoints with a `->`
between them. That is deliberate: an arrow reads as direction where a bar reads as a boundary, and no
session identity can contain `->`, so the field still splits mechanically. It does mean the addressee is
matched on the **arrow**, not on a bar, which the recipes below reflect.

**Separate the fields with `|`, not with whitespace.** Bars survive what spaces do not: a double space
is invisible in a diff, collapses to one under any reflow, and stops being a separator the moment a
subject contains two spaces of its own. With bars the header is parseable positionally — `awk -F'|'`
for the fields, then split field 2 on `->` for the two endpoints — which is what lets a watch or a
reader triage on it mechanically rather than by eye.

Anchor an addressee match on the arrow, and allow for a list, because **naming primary recipients and
still broadcasting is the common case, not an edge one**: `-> WP5, WP4, ALL` says "chiefly you two,
everyone else for the record", which is worth writing. So the matcher is `grep -E -- '-> [^|]*WP4'` for
one session and `grep -E -- '-> [^|]*ALL'` for broadcast. **Not** `grep -- '-> ALL |'`, which only sees a
broadcast with no named recipient: measured on a live log, it caught 3 of 8 and missed every list form,
one of them written by the session that had published the recipe four minutes earlier. And
**not** `grep '| WP4 |'`, which matches no header at all, the addressee sharing its field with the
sender; it does match prose *about* the recipe, so a bare count on it is not zero and reads as a
contradiction.

**A pass rule must fail open on a header it cannot parse, and absence of an addressee means everyone,
never nobody.** This is the rule that matters most in this whole section, because breaking it is
invisible to *both* parties at once. A session mid-close, or one that has not read the notice, keeps
writing the previous format; a filter written for the current one then drops every block it sends, and
neither side learns anything. The writer sees a post that went through. The reader sees a quiet log.
"That peer has gone quiet" and "that peer has finished" look identical, and one of them means their
release never reached you.

An `INVALIDATES` announcement does not save you, and it is worth being clear about why: it is broadcast
traffic, so it cannot reach a session that has stopped reading broadcast traffic, which is precisely the
session whose format is stale. **The reader carries this, not the writer.** Concretely: key the pass rule
on `^## ` and the tokens you care about anywhere in the line, never on field positions; treat a header
with no `->` as addressed to you; and cover every separator the log contains rather than only the current
one, since append-only guarantees the old shapes stay.

That is the same rule as *a check must refuse, rather than report clean, when its input is empty*, one
level out: an absent addressee is an empty input, and concluding "not for me" from it is reporting clean
on nothing.

**A filter has two rules, and both must fail open, which means opposite things for each.** Stating only
the pass side is a trap, because the suppress rule is where a session is looking at its own blocks and
therefore not thinking about what else could match. Ask instead: **when this rule misfires, who pays?**
A pass rule that misfires loses a message, so it must fail towards showing you too much. A suppress rule
that misfires *also* loses a message, so it must fail towards showing you something **twice**, never
towards hiding something once. Write each so the cheap outcome is the one that happens.

Concretely, the discriminating case is a **loose** self-suppression. `^## .* \| WP1 -> ` matches that
string anywhere in the header, so it silently drops a *peer's* block whose subject quotes the form
`| WP1 -> ALL |` — and a block about header formats is exactly the traffic you can least afford to lose.
Anchor self-suppression on the sender field, `^## [^|]*\| *WP1 *->`, so it can only match where the
sender actually is.

**Your identity is your task, not an ordinal.** A session running this skill has exactly one job for
its whole life, the entry point and its argument, so that is already a name: `WP3`, `WP13.3`,
`bootstrap`, `retrospect`. Do not allocate yourself a `session-A` or a `session 2`.

The first reason is the one that matters. **An ordinal has to be allocated, and allocating anything
between sessions that cannot see each other is the problem the log exists to solve.** Two sessions
arriving at once both read the log, both see no `session-A`, and both take it — the same race as
picking "the lowest free WP number", which this skill already handles by requiring a claim. A
task-derived name needs no allocation, because the user made the names distinct when they gave each
session a different task. The second reason is that it carries information:
`CLAIM: numerics/data | WP13.3` tells a reader whose work is at stake, where `session-B` tells them
nothing and has to be looked up. If two sessions genuinely share one task, disambiguate on whatever
differs between them rather than by counting.

**Address each block to whoever must act on it, and reserve `ALL` for what changes what an uninvolved
session does** — a format change, a log-integrity alert, an arrival or departure, a claim. Findings between
two sessions are addressed to those two.

**The field collapses without that sentence, and it collapsed within ninety minutes of being introduced.**
Measured on the log it was introduced in: **57 of 57 blocks named `ALL`**, from every session including the
one that wrote the rule, and not one was addressed to named sessions alone. A filter keyed on `ALL` then
passes everything, which is the state before the field existed, and the promise that a reader can see
`-> WP4` and stop describes something that never once happened.

The cause is an incentive, not a wording. Omitting `ALL` risks a peer missing something; including it costs
peers a read. **The cost of over-addressing falls on others and the cost of under-addressing falls on
you**, so everyone maximises. The counterweight is already here and must be stated beside the rule or it
will not be applied: **because a pass rule fails open, under-addressing costs a late read, not a lost
message.** A session that should have been named still sees the block, through its own filter's fail-open
branch or through `INVALIDATES`, `ALERT` and `DONE` passing unconditionally. Address narrowly; the
machinery is built so that being wrong about it is cheap.

**Do address every block to somebody, and write `ALL` rather than leaving the field blank.** Without an addressee
a log is a broadcast, but a watch notification *arrives* like a message to you, and that asymmetry
misleads: a session that has recently posted reads the next block's "your" as its own. It happened
here. A block from one session to another, twenty-three minutes and six hundred lines after a third
session's unrelated post, was read by that third session as a reply to itself, and produced a confident
correction of an attribution that had been right all along.

An earlier version of this file tried to fix that by asking authors to *name the peer in the subject*.
That was too weak: prose in a subject line is optional, cannot be matched positionally, and its absence
is ambiguous between "for everyone" and "the author forgot". A field is none of those things. `-> ALL`
is a statement; a blank is a guess.

Two consequences. **"You" in a block means its addressee**, so second person is now safe and needs no
archaeology — the earlier rule about resolving "you" from the block above is superseded. And **triage
becomes free**: a session sees `-> WP4` and stops, without reading a word of the body. That is what
makes wide perception affordable, and it is the real answer to the complaint that a watch which emits
everything costs a read per event to conclude the event was not yours. The cost was never perception;
it was a header you could not triage on.

**Changing a separator invalidates every filter written against the old one, and those filters live
in other sessions' processes where you cannot see them.** Announce a format change as `INVALIDATES`, and
say which shape moved. This is not hypothetical: the block announcing the bars broke the self-echo
suppression in three sessions' watches at once, including its author's.

The direction of the breakage decides how bad it is, and it is worth checking which kind you have.
A **suppression** rule written against the old shape fails **open**: it stops matching, and you get
noise you can see, which is what happened. A **pass** rule written against the old shape fails
**closed**: it stops matching, the watch goes silent, and silence is indistinguishable from a quiet
log. So a session whose filter selects on header shape should cover **both** shapes, since append-only
means the old blocks stay as they are and only new ones carry the new separator.

**Append with `>>`, and never read-modify-write a log that peers are appending to.** Reading the whole
file, transforming it in memory and writing it back is a lost-update race: any append that lands between
your read and your write is **erased**, silently, and the erased block belongs to someone else. Nothing in
the log will show it, no watch will alert on it, and the author will believe their message went through.
The append-only rule is usually explained as "never truncate"; this is the sharper form, because a
read-modify-write does not look like a truncation and can shorten the file by exactly one peer's block.

**Compose the finished line before it touches the file, rather than writing a placeholder and substituting
it afterwards.** A placeholder is a read-modify-write with a second failure mode on top: the log is a
corpus that discusses its own syntax, so the moment a peer writes prose *about* your placeholder, a
first-occurrence substitution retargets onto their text. Build the header with the timestamp already in
it, `printf '## %s | ...' "$TS"`, and append once.

**And verify the append by reading it back, because a substitution step that matches nothing still exits
0.** One session's posting helper printed `posted at <time>` unconditionally, after a replacement that had
silently matched nothing, and the malformed header sat on disk being read by four peers while its author
believed it was fine. This is the empty-input rule again, now in a tool rather than a checker: report what
you read back, never what you intended to write.

**The log is ordered by append, not by timestamp, so the timestamps are not monotonic.** Two sessions
posting within the same minute land in whichever order the writes complete, and a slow write can land
minutes out of order: one live stretch reads 13:44, 13:47, 13:45, 13:45, 13:46, 13:47. Two consequences,
and both have bitten. **Attribute a body line to the nearest preceding header by file position**, never
by walking forward to "the next later timestamp", which skips whole blocks. And **never infer that two
adjacent blocks are a reply and its answer** from their order alone; the addressee field is the only
evidence of who a block is for.

**A coordination log is a corpus that discusses its own syntax, so every parser over it will eventually
match its own documentation.** This is the single most productive source of defects the mechanism has,
because the traffic that quotes the syntax is the traffic *about* the syntax, which is the traffic you
least want mis-handled. Four distinct instances, all live: a published `grep` recipe returned matches
that were prose about the recipe; a quoted claim line was replayed as a real claim; a subject containing a
header form was silently dropped by a loose self-suppression; and a posting helper substituting its first
match retargeted onto a peer's prose *about* that placeholder, which is the same failure on the **write**
path and the one that damages other people's text rather than your own reading. The defence is positional
anchoring — `^## ` for a header, `^CLAIM:` for a claim, the sender field for self-suppression — plus the
next rule, which is the writer's half.

**Never reproduce a `CLAIM:` or `RELEASE:` line at column 0 in a block that is not making that claim.**
Indent it, inside the fence, when quoting. This is the writer's half of the owner-field rule above, and
it stays load-bearing rather than becoming redundant: it is what protects a log whose lines carry no owner,
which means every line written before the field was specified and every line from a session that has not
adopted it, and a format change cannot reach a session that has stopped reading broadcast traffic. Where
the owner is present, a quotation is harmless; where it is absent, this habit is the only defence. The
protocol puts claims at column 0 in the body so they can
be found without parsing prose, which means a quoted one is indistinguishable from a real one and
transfers the apparent claim to whoever quoted it. That is not hypothetical: one session quoted another's
claim in order to ask a question about it, and a positional parse then reported the *asking* block as
holding all five paths. The misattribution had already travelled one hop by then, because the same two
mechanisms compound: the quoted claim was itself picked up from the wrong header by append order.

**Put the message type in the header, in capitals** — `ARRIVED`, `INVALIDATES`, `CLAIM`, `RELEASE`,
`FINDING`, `CORRECTION`, `NOTE`, `DONE`. A watch emits header lines and nothing else, so the type is
what tells a reader whether an event needs acting on now or reading later. A header without one costs
a file read to find out.

### Reading it

Two different reads, for two different needs:

- **The lock set** — grep or `awk` the *whole* log and replay `CLAIM:`/`RELEASE:` with last-one-wins.
  Output is a handful of paths whatever the log's size, so it costs nothing and cannot miss a block you
  skipped. Three details decide whether the replay is right, and each has been got wrong in practice:

  **Write the owner in the line, and replay per `(owner, path)`:**
  `CLAIM: paper/refs.bib, paper/paper.tex | WP5`. Strip the trailing `| owner`, then split the paths on
  commas. This field is not new — it is in this file's own example above — but it appeared only inside a
  paragraph making a different point, while this recipe said nothing about it, so every session wrote
  `CLAIM: <paths>` and the replay had no owner to key on. **An example is not a specification**, and a
  field documented only by illustration will not be used.

  **Replay per path, never per line.** A multi-path claim is not an atom, and string equality between a
  claim line and a release line is a coincidence the format never guaranteed: a claim of five paths
  released one at a time stays "held" forever under a line-keyed replay. For an ownerless legacy line,
  fall back to the nearest *preceding* header by file position, since append order is not timestamp order.

  **Why the owner field is load-bearing rather than decorative, and why "the state of the log" is not a
  thing.** A quoted claim line makes the quoting session an apparent claimant, and what that costs
  depends entirely on how the replay keys. Measured on one live log, five paths, genuine claim eleven
  minutes before the quotation:

  ```
  keyed on path alone        the quoting session holds all five; the genuine holder holds NOTHING
  keyed on (sender, path)    two claimants per path; the genuine claim survives intact
  ```

  Both parsers are defensible and they disagree about who holds the files, so **the lock set is a
  property of the log *and* the parser, never of the log alone.** Say which replay produced a holder
  before acting on one. Reporting one parser's output as "the state" is the same defect as reporting a
  match count without naming what was counted, one level up.

  With the owner in the line, both pathologies go away and so does an exotic-looking rule. A quotation
  replays to its true owner, so it creates no claimant at all; and where a spurious claim does exist, its
  author simply releases it, because the release keys to `(author, path)` and cannot touch
  `(true holder, path)`. **The apparent property that a phantom claim is irreparable by its author, and
  that only the true holder can repair it by re-asserting, is an artefact of the missing owner field, not
  a property of append-only logs.** It is worth knowing only because a log written without the field has
  it.
- **The prose** — read incrementally from an offset stored in a **file** in your scratchpad. Context is
  exactly what a long gap destroys, so an offset held in context is worthless after one.

Incremental reading is only sound *because* the log is append-only. That is the real reason for the
never-truncate rule.

**Only reading advances the offset. Writing never does.** After appending your own block the file is
longer, and setting the offset to the new end marks as read everything a peer appended since you last
looked. The same failure has a second form: reading from a *pattern match* rather than from the stored
offset, then declaring the file read, which silently swallows the whole prefix between the offset and
the match. Both are silent, and both were committed in practice, in two projects, on the same day.

**General form: any step that advances an offset without delivering content converts "not yet seen"
into "already seen", silently.** Writing is one such step, a pattern match is another, and a third is
a **watch that emits headers and advances past the prose between them** — which is what the
emit-headers-only rule below produces if that same position is also the read offset. It then means
*scanned*, not *read*.

So **keep a scan position and a read position as separate quantities**; one variable holding both is
always lying about one of them. **Emit the line number with every header**, so an event carries the
locator its prose needs after the position has moved past it. And when in doubt, enumerate
`grep -nE '^## '` — a fact about the file, where an offset is only a claim about your own attention.

**Arm the watch to replay from the beginning, not from the current end.** Output is bounded by the log's
size once, and it costs a session nothing it was not already required to read; arming at the end buys a
few lines of quiet at the price of a silent prefix. This is the form `SKILL.md` states as an action at
registration, and the reason follows.

**Arming a watch is not reading.** A watch started at the current end of file has just set an offset
to a position nobody read, and it will report only what arrives afterwards. Every block already in the
file is then invisible to that session forever, silently, and this is the form the author of this
paragraph committed: a watch armed at line 1614 declared 1613 lines read, including the block that
prompted the finding being written up. Arm it at your **stored** offset, not at the end; if you have
no stored offset, enumerate first and set it deliberately.

**When you doubt the offset, do not repair it. Enumerate the headers.** `grep -nE '^## '` over the
whole file lists every block, is complete *regardless* of what the offset claims, costs one command
whatever the log's size, and does not depend on the mechanism that just failed. Then account for the
blocks you have actually seen. This is the same argument the lock set rests on, bounded output over the
whole file, and it applies to headers for the same reason — which is why the header format is worth
keeping strict enough to grep.

The distinction is worth stating plainly: an offset is a claim about your own past attention, and a
header count is a fact about the file. When they disagree, the file wins.

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

**Better than either form: have the watch touch a heartbeat file each poll, and read its mtime.** A
process table tells you whether *a process matching a pattern* exists, which is not the question; it also
over-counts, because the shells that launched and check the watch carry the pattern in their own command
lines. Measured: three matches for a single watch, and the checking shell was not among them. A heartbeat
answers the actual question, and it is the difference between inferring liveness and measuring it.

Write it under `docs/coordination/heartbeat/<your identity>` on every poll of the loop you have
already got, and **write the poll interval into it** (`printf 'poll=%s\n' "$POLL" > "$HB"`) rather
than touching an empty file. Then a peer's test is `mtime` against a small multiple of that interval,
which is **measured rather than declared**.

The interval has to be **in** the file: an empty heartbeat leaves the peer — the party this rule is
written for — with no interval to read and a threshold to invent, which is the declared cut wearing a
measured cut's clothes.

**Never commit a heartbeat, and exclude the directory as part of the same step that creates it.**
Two lines beside the `mkdir`, idempotent, so a project acquires the rule whenever it acquires the
mechanism:

```bash
mkdir -p docs/coordination/heartbeat
grep -qxF 'docs/coordination/heartbeat/' .gitignore 2>/dev/null \
  || printf '\n# Liveness heartbeats: mtime is the whole signal, and git stores no mtime.\ndocs/coordination/heartbeat/\n' >> .gitignore
```

It belongs *here* rather than only in `bootstrap.md`, because coordination is retrofitted into repos
bootstrapped long before it existed: the directory is created by whichever session arrives first, and
a rule living only in bootstrap reaches none of them. Under the exclude-only and symlink storage
modes `docs/coordination/` is already excluded wholesale, so this matters exactly when the log is
tracked.

**Why committing one is worse than untidy:** a heartbeat is 0 bytes, so its entire signal is the
**mtime** — and git stores no mtime. On clone every file gets the *checkout* time, so a committed
heartbeat comes back looking freshly touched and asserts that a long-dead session is alive. The
fail-open property below holds only while a *present* heartbeat cannot lie.

**It fails open, and this is the half that makes it safe.** No heartbeat means *unknown*, never *dead*: a
session may simply not be running a watch, which was true of two sessions out of five on the day this was
written. So a missing file licenses nothing. What a *stale* heartbeat licenses is the reversibility
judgement below, with better information than a guess.

It also composes with `sign-off` without a special case. A session that signs off stops its watch, so its
heartbeat goes stale by construction, and the stale heartbeat and the `DONE` block agree. A session whose
window was closed leaves the stale heartbeat and no block, and **that difference is exactly what
distinguishes a departure from a disappearance** — which is the thing no amount of reading the log could
otherwise settle.

**The thing to regress is whatever you just added.** The rule reaches further than predicates: a
*fixture* is not coverage until something fails when the fixture goes unseen. One session found a blind
spot in its own checker, added a specimen from the blind class, and stopped — and measured that with the
specimen present and nothing asserting on it, the broken checker still exited 0. **A specimen nobody
checks is decoration**, and it is the guarded defect one level up, since the fixture's presence is
indistinguishable in the output from the fixture being examined.

The naive predicate returns 1 for a pattern that matches **nothing**, so it is provably incapable of
reporting "not running" — shown in one command, at the moment the check is written, with no reasoning
required.

**Do not solve a narration problem by reducing perception.** A watch that emits every peer header is
tiring: each event costs a read to conclude it was not yours, and it is tempting to narrow the filter.
That is the wrong layer. The complaint is that you are *reporting* too much; the fix is to report less,
not to see less. **Seeing everything and forwarding little is strictly better than seeing little**,
because the judgement about what matters is then made by you with the evidence in hand rather than by a
grep pattern without it. One session narrowed its watch for exactly this reason, then widened it back
within twelve minutes on exactly this reasoning.

If you do narrow it anyway, two conditions, and the first was learned by its violation. **Verify the
guarantee underneath is actually running before you thin the convenience on top of it.** The same
session justified narrowing on the grounds that read-on-return was "unchanged" — while having treated
one of nine mid-turn user messages as a return point. It was relying on the watch as its sole delivery
mechanism, calling it best-effort, and thinning it. An assumed guarantee is not a guarantee, and this is
the *stated premise* of a rule going unchecked rather than a number going unchecked. Second, **publish
what the filter drops, not only what it passes**: an unstated filter is indistinguishable from a peer
who has gone quiet, and quiet is indistinguishable from absent. Keep `INVALIDATES` and any
log-integrity `ALERT` passing unconditionally from anyone, and `DONE` with them: a departure is precisely
the message whose loss leaves you unable to tell a finished peer from a silent one, and one close-out has
already been lost that way. Test the filter on a fixture in both
directions before arming, and confirm the old watch is gone and the new one up, or you have two.

Publishing it means saying so in the log, concretely: *to reach me, address a block to me or to `ALL`,
claim one of these paths, or use `INVALIDATES`; anything else I will see late.* The author of this
paragraph narrowed a watch to suppress its own posts and never published that, so a peer wondering why a
message had not landed had no way to find out.

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

**`ARRIVED` is a trigger, not only an announcement: when one appears, every live session welcomes the
newcomer and says what it inherits.** This is the one duty in this file with a genuine external trigger,
which is why it is worth having as an obligation rather than as advice. A block appears in the log, your
watch surfaces it, and the response is owed. Nothing has to be remembered.

Say what the newcomer would otherwise reconstruct or trip over. In practice that is three things: what it
can rely on, and **how you know**; what of yours it is about to stand on, especially in paths it just
claimed; and any trap in a file it claimed that the file does not advertise.

**Mark each statement demonstrated, observed or recalled, and never write "you can rely on this without
re-checking".** A handover is inheritance, and inheritance is not evidence. Telling a newcomer it need not
check is the one thing an arriving session must not be told, because it is the party with no context and
therefore the least able to notice that a statement has gone stale. Marking how you know moves the trust
decision to the reader, where it belongs, and costs a word.

This was measured on the first arrival after the duty was written. Of the handover statements that session
received, **two did not reproduce**, and one of them mattered: an instruction to remove a passage from a
reusable file, where the passage was not in the file, so the real task was to avoid introducing it. A
different work package, reached only by checking. The session that found it had run a probe with a control
that fired, and reported both results as *demonstrated, not observed* — which is the distinction this rule
now asks for, and it came from the sessions using the log rather than from this file.
One session warned an arriving session that a file it had claimed one minute earlier carried scaffolding a
later work package deletes, which is exactly the class: not a durable conclusion, not a diff-review aid,
just a thing one live session knew and the next would have hit.

**If you have nothing, say so in one line.** That is not a courtesy, it is information: the newcomer can
read the population off the log, so a short "nothing from me" distinguishes *we do not interact* from
*that session has not seen you yet*, and silence cannot.

**Do not put this in a session's departure instead.** It is tempting, because a departing session
obviously has knowledge to pass on, and it is wrong in both directions: a session signing off before any
successor exists has nobody to write to, and a session arriving when nobody happens to be leaving gets
nothing, which is the common case and the one where context is most needed. Measured on the arrival that
produced this rule: four handover blocks inside two minutes, and **two came from sessions that were not
going anywhere**. The two that sat inside sign-offs did so by coincidence of timing, and reading that as a
departure duty inverts the trigger.

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

**Without a heartbeat there is no reliable way to tell whether a session is still alive**, and with one
there is, for any session running a watch: see the heartbeat rule above. What follows applies to a session
that has none, and to the interval before a stale heartbeat is old enough to act on. Directory mtimes are
not a liveness signal — a live session's own directory can read days stale, because a directory's mtime
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

**Never author a claim in the same tool call that produces its evidence.** Measure, read the result,
then write. When a command and the sentence reporting it go in one call, the sentence was composed
before the output existed, so it is written from *expectation* and nothing forces a reconciliation
afterwards. This is structural rather than a lapse of attention, which is why "read your own output"
is not a usable rule and this is: splitting the two makes writing-from-expectation hard instead of
merely discouraged, and it costs one extra tool call.

It explains a pattern otherwise hard to account for. One session's shipped *guards* were all sound
while several of its *reports* were false, and the difference was that every guard had been forced
into a separate step and read before it was believed, while every false report had been co-authored
with its own evidence. Another published a byte count of `1722` where the file said `1721`, having
composed the sentence from a measurement taken before an edit it then made in the same breath.

**A count is not a measurement until what was counted is named.** Report the pattern beside the number,
because the number is produced by the pattern and the pattern is the part that can be wrong. `6 matches`
is not evidence; `6 files containing all three values` is. The failure never looks like a failure: every
one of these was a real count of the wrong thing, reported with confidence.

Promoted from a candidate on six instances across four sessions and five mechanisms: a regex matching a
prefix shared with a different quantity; a header count taken against a different total; a total that moved
because newly added fixtures cited real keys; an unanchored pattern counting a document's prose *about* a
search recipe as if it were data; the same mechanism again on a different pattern; and, most general of
all, **a real count of a real thing reported as a property of the object rather than of the instrument** —
a claim replay keyed on path alone and one keyed on owner-and-path return different holders, both
defensible, so there is no fact of the matter about "who holds the file" that is independent of the parser.
Nothing was miscounted in that last one. The error was the omitted clause naming which instrument produced
it.

**A check must refuse, not report clean, when its input is empty.** A verification that finds nothing
must distinguish *the specimen is clean* from *there was no specimen*, and treat the second as fatal
rather than passing. An empty specimen, an empty reference, a glob that matched no files, a pattern that
matched nothing because the text moved: each produces the same silence as a genuinely clean result, and
no attention to the output recovers the difference, because the difference is not in the output.

Promoted from a candidate on four instances across three sessions and two distinct mechanisms: a
phrase-based sweep blind to wrapped text, an empty bibliography file, an empty comparison bucketed as
non-fatal, and a regex requiring an uppercase initial that silently skipped three bibliographic keys,
including the one carrying a register's most consequential attribution, and then reported clean. **Every one reported success because of what it could not see.** The fix each time was
the rule below: run the check in the state where it must fail.

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
