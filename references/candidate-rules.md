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

## A check must refuse, not report clean, when its input is empty

**Would read:** a verification that finds nothing must distinguish *"the specimen is clean"* from
*"there was no specimen"*, and refuse in the second case. An empty specimen, an empty reference, a glob
that matched no files, a grep pattern that matched nothing because the file moved: each yields the same
silent pass as a genuinely clean result.

**Evidence, one project.** In `project-2026-inputshifts` an audit floor was reported as having fired
when the underlying query had returned 43 rows rather than 0; the check could not tell the two states
apart. It sat beneath the single sentence that `docs/LITERATURE.md` labels its strongest claim, so the
consequence was not hypothetical.

**Why it is mechanism and not carelessness:** the failing and passing states are indistinguishable in
the check's own output, so no amount of attention to that output recovers the difference. It is the
same family as *a check's characteristic failure is to confirm the hypothesis it was testing* in
[parallel-sessions.md](parallel-sessions.md), and would likely be folded in beside it.

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

## A count is not a measurement until what was counted is named

**Would read:** report *what* was counted alongside the number, because a count is produced by a
pattern and the pattern is the part that can be wrong. `6 matches` is not evidence; `6 files containing
all three values` is.

**Evidence, two sessions, one project pair.** A constant was reported as appearing under four names
when it appeared under three, because the pattern matched a *prefix* shared with a different quantity;
the counter had counted names matching its regex rather than names carrying the value. Independently, a
header count of 45 was accepted where the file held 48. Both counts were real numbers of the wrong
thing.

**Caveat on promotion:** the two instances are from sessions that had been talking to each other for
hours, so they are not independent in the sense the criterion requires. A genuinely separate occurrence
is still wanted.
