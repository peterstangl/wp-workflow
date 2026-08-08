# Candidate rules: judged real, not yet general

*Part of the `wp-workflow` skill. Read this from [retrospect.md](retrospect.md) when triaging lessons;
it is not needed by any other entry point.*

A holding area for lessons that met the *mechanism, not regret* test in `retrospect.md` — they name
something a session could not have avoided by trying harder — but that have been seen in only one
project. Shipping every one into the reference files would grow them for a reader who may never meet
the pattern; discarding them loses work already done.

**This file exists because the alternative failed.** Three candidates were once "deferred" by recording
them in a project's coordination log. That log was excluded from version control and disposable, and
the condition attached to them, *"until a second project meets them"*, was unevaluable by construction:
a second project cannot read another project's log. A deferral with no mechanism for being revisited is
not a deferral. This file is outside every repository, survives every project, and is read by exactly
the entry point that would promote its contents.

**Promotion criterion.** A candidate becomes a rule when a *second, independent* project or session
meets it — not when it recurs in the project that found it, since one project's habits produce
correlated failures. Record the second instance here first, then move the rule into the reference file
it belongs to and delete the candidate. **Demotion is also allowed**: a candidate contradicted by later
evidence should be struck with the reason, since a stale candidate is a recorded constraint and decays
like any other.

---

## ~~A check must refuse, not report clean, when its input is empty~~ — PROMOTED

Moved into [parallel-sessions.md](parallel-sessions.md) on 2026-08-08. Promoted on **four instances
across three sessions and two distinct mechanisms**, which satisfies the session clause of the
criterion even though all four arose in one project: a phrase-based sweep blind to wrapped text, an
empty bibliography file, an empty comparison bucketed as non-fatal, and a regex requiring an uppercase
initial that silently skipped three bibliographic keys and reported clean. The strongest of the four is
the last, and not for the reason first recorded here: it was **already shipped in the committed code of
the very session that had identified the class**, undetected, and reported clean four times in that
session's own status updates before a *different* session found it. A class its own discoverer is
simultaneously committing is general, not local.

## A closing block should carry the lesson that would change another session's practice

**Would read:** a session ending should name what it got wrong in a way another session can act on, not
only what it did. The value is in the *mechanism* it misdiagnosed, since that is the half that travels;
what it built is already in the repository.

**Evidence, one instance.** One sign-off carried a section titled "one thing I got wrong that is worth
carrying, since it cost four sessions time": it had found a real symptom, attached the first cause that
fit, and broadcast the mechanism, which three sessions then failed to reproduce. It also recorded that it
repeated the same error twice more within the hour, once *after* writing that very paragraph.

**Why it is not yet a rule.** One instance. A second closing block the same day carried a generalised note
for later readers, which is a different thing, and a third carried none. Held here rather than added to
[sign-off.md](sign-off.md) because a required confession invites performance, and because the promotion
criterion this file sets exists precisely to stop a single vivid instance becoming procedure. Promote it
if a second session, unprompted, ends with a misdiagnosed mechanism rather than a summary.

## A delegated summary is evidence; a delegated judgement is not

**Would read:** what a subagent, a peer or a document *reports having seen* can be treated as evidence and
checked cheaply; what it *concludes* cannot be inherited at all. Read the source before acting on a
judgement, and before repeating one.

**Evidence, three instances in one day, two of them at one remove.** A session sent an exploring subagent
over a draft and got back a confident, plausible claim that the text contradicted itself on a physics
point; opening the two passages showed it consistent, and the misleading thing was a subsection title. A
second session described the contents of a file it had never opened, inferring them from a one-line
pointer, and published that as a finding. A third session, the author of that file, then **repeated the
characterisation to its own user as better than its own analysis, without opening the file it had
written.**

**Why it may be mechanism rather than carelessness:** a judgement arrives in the same shape as an
observation, with the same confidence and none of the provenance, so nothing in the report distinguishes
them. That is the same family as *a count is not a measurement until what was counted is named*, and if it
promotes it probably belongs beside it.

**Held because the three instances are one project on one day**, and because the strongest of them is the
skill author's own, which makes it a class this file's keeper is inside rather than observing. Promote on a
sighting in a second project.

## `$?` after a pipe reports the wrong command

**Would read:** in a pipeline, `$?` is the exit status of the *last* command, so a guard reading it
tests the wrong thing. Use `PIPESTATUS`, `set -o pipefail`, or avoid the pipe. More generally, a
harness that checks a status must be shown to distinguish success from failure, not merely to report
success.

**Evidence, one project.** A probe in `project-2026-inputshifts` reported a guard as having held when
the guarded command had failed upstream of the pipe. The same session's equivalent Python check was
sound, because `subprocess.run` exposes `returncode` on an object with no shell between the guard and
the reading of its status.

**Relation to a shipped rule:** *run the test in the state where it must fail* would have caught it, so
this may be a worked example of that rather than a rule of its own. That is the judgement to make on
promotion.

## ~~A count is not a measurement until what was counted is named~~ — PROMOTED

Moved into [parallel-sessions.md](parallel-sessions.md) on 2026-08-08, on the judgement recorded below.
The evidence is kept here because the promotion turned on *mechanism* diversity rather than on session
independence, which is a reading of the criterion the next promotion should either follow or overturn
deliberately.

**Would read:** report *what* was counted alongside the number, because a count is produced by a
pattern and the pattern is the part that can be wrong. `6 matches` is not evidence; `6 files containing
all three values` is.

**Evidence, two sessions, one project pair.** A constant was reported as appearing under four names
when it appeared under three, because the pattern matched a *prefix* shared with a different quantity;
the counter had counted names matching its regex rather than names carrying the value. Independently, a
header count of 45 was accepted where the file held 48. Both counts were real numbers of the wrong
thing.

**Third instance, 2026-08-08, self-identified by one of the same sessions and by a different
mechanism:** a resolved-key total moved from 97 to 98 not because the bibliography changed but because
newly added selftest fixtures cite three real keys. The reporter named it as an instance of this very
rule — the total is a property of the file *plus what you count*, and it moved for a reason unrelated to
the quantity it appears to measure.

**Fourth instance, 2026-08-08, a different session and a fourth mechanism: a self-referential corpus.**
A published `grep` recipe was found to be broken by a peer who tested it. Verifying the fix, the broken
pattern returned **3** matches where the claim was that it matched nothing — and all three were prose
*discussing the broken pattern*, including the sentence that published it. Zero were block headers, so
the claim was true of the object it was about and false as a bare number. Naming what was counted was
what separated them, and the correction was about to ship with the unqualified "matches nothing" in it.

**Fifth instance, same session, same mechanism as the fourth.** A broadcast-coverage count was reported
as 5 of 12 and was 3 of 8, the pattern being unanchored and so counting prose about the recipe as headers.
It repeats a mechanism rather than adding one, which is worth recording precisely because a repeat inside
one session is what the promotion criterion is designed to discount.

**Sixth instance, same session, and a fifth mechanism that is the most general form yet: a real count of
a real thing, reported as a property of the object rather than of the instrument.** A claim replay keyed
on path alone returned a holder; keyed on `(owner, path)` it returned a different one. Both parsers are
defensible, so there is no fact of the matter about "who holds the file" independent of the parser, and
the output was reported as the state of the log. Nothing was miscounted. The error was the omitted clause
naming which instrument produced it.

**Promotion judgement.** This now stands at six instances across four sessions and **five** distinct
mechanisms: a regex matching a shared prefix, a header count against a different total, a total moved by
newly added fixtures, a corpus containing prose about its own search pattern, and an instrument-dependent
result reported as instrument-independent. That meets
the standard the first promotion in this file was held to, and the independence caveat above is
answered on mechanism rather than on session identity: four mechanisms cannot be one project's habit.
**Proposed for promotion, not promoted** — a candidate is moved by the entry point that reads this file,
with the user's approval, and shipping it silently would be the defect the file exists to prevent.
