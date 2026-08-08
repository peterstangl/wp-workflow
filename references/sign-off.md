> *Part of the `wp-workflow` skill. The router, the invariants and the coordination rules are in `../SKILL.md`; the other entry points are alongside this file in `references/`.*

## Entry point: `sign-off`

Goal: end a session deliberately, leaving nothing running, nothing claimed, and no ambiguity for a peer
about whether you finished or went quiet.

### Step 0 — the trigger, which is narrow on purpose

**Run this only when the user says "sign off" or "sign-off".** Nothing else triggers it: not "are you
done", not "can I close this", not "is everything finished", not a background event, and never your own
judgement that the work looks complete.

A status question and a sign-off are different acts. **"Are you done?" is a question whose answer may be
no**; sign-off is an action that is only valid once the answer is yes. Routing the question to the action
means a status enquiry begins a termination procedure, and worse, it invites reading a question as
authorisation to end. Answer questions as questions: run the completion criteria below, report what is
outstanding, change nothing.

### Step 1 — verify, and refuse if you cannot

Work through both lists and **report measured results, never assertions**. Then:

**Refuse to sign off if any item is incomplete *or* cannot be determined.** "I could not tell whether the
watch stopped" is a refusal, not a pass. A sign-off's entire value is that it was checked, so a check that
cannot distinguish done from unknown must fail closed, which is the empty-input rule applied to the one
procedure whose whole job is confirming. On refusal, name exactly what is outstanding and stop; do not
sign off partially, and do not offer to sign off "apart from" something.

**Completion criteria, by the entry point this session ran:**

| entry point | done means |
|---|---|
| `bootstrap` | `CLAUDE.md` and `docs/PLAN.md` written, every WP shipped `[ ]`, nothing implemented |
| `implement <WP-id>` | Step 5 finished: box `[x]`, that WP's verification actually run, both files updated, size check done |
| `amend-plan` | `docs/PLAN.md` updated and the WP numbered, nothing implemented |
| `archive-plan` | archive file written, live file compressed, link in place |
| `retrospect` | edits applied, size budget reported, changes summarised for the user |

**Coordination criteria, for every entry point:**

1. **Claims.** Replay the log **both** owner-keyed and path-keyed, and show zero paths held by this
   session under **both**. Two replays because they disagree, and the disagreement is the point: a lock
   set is a property of the log *and* the parser.
2. **Background work.** Enumerate every watch, monitor and background shell this session started. This is
   an enumeration, not a recollection.
3. **Durable conclusions.** Anything a future reader will need is in `CLAUDE.md`, `docs/PLAN.md` or a
   design document, with a pointer left in the log. A conclusion that exists only in a chronological log
   is one nobody will find.
4. **Open threads.** Questions you asked that nobody answered, questions a peer asked that you never
   answered, and any criterion you published ("I will post again if…") that you are now withdrawing.
5. **Working tree.** Report what is uncommitted, with the two handover lists: changed on purpose, and
   changed as a consequence. **Never commit** (invariant 1).

### Step 2 — confirm with the user

Present the verification and ask. Only the user knows whether they are finished with this session, and
that is the whole reason this entry point exists rather than a session deciding for itself.

### Step 3 — end, in this order

The order is counter-intuitive and load-bearing. **Background work stops last**, because until the
terminal block is on disk you must still be reachable: a peer may object to a release, or ask you to stay.

1. **Post the terminal block**: type `DONE`, addressed, releasing every claim by owner with the paths
   spelled exactly as claimed. A release whose string differs from its claim never clears it.
2. **Verify it on disk** by reading the header back. A post that reports success without being read back
   is a report of intent.
3. **Say what your silence now means.** This is the sentence peers cannot infer: that you have stopped
   looking, not that you have found nothing. Without it, "finished" and "gone quiet" stay
   indistinguishable, which is the ambiguity the whole block exists to remove.
4. **Stop each watch and background shell**, then **verify each is gone** rather than assuming a stop
   succeeded.
5. **Report to the user** what was stopped, what was released, and what remains for them to commit.
6. **End the reply with the closing line below, verbatim, as its last line.**

### The closing line

A completed sign-off ends with exactly this, on its own line, as the final thing in the reply:

> `SIGN-OFF COMPLETE — nothing I started is still running, and you can close this window.`

**It is fixed wording in a fixed position so it can be recognised without being read.** The user asks for
a sign-off in order to learn one thing: whether the window is safe to close. A sentence that is phrased
differently every time has to be read and interpreted, which puts the reader back in the position of
judging from prose what a procedure was supposed to have settled.

**Three rules make it worth trusting, and all three are about when it must be absent.**

**Never write it unless every step above actually completed and was verified.** The line is not a summary,
it is a guarantee: it asserts that nothing this session started is still running, and it may be emitted
only after each stop has been confirmed by looking. If one watch could not be verified gone, there is no
closing line, however small the doubt.

**Never write it on a refusal, and never write a hedged variant.** No "signed off apart from", no "you can
probably close this". A refusal ends with what is outstanding and nothing else. Because the line appears
in no other circumstance, **its absence is itself unambiguous**, and that only holds if the rule is kept
exactly.

**Never reproduce it while explaining the procedure.** A log is a corpus that discusses its own syntax and
so is a chat: a session describing sign-off, quoting this file, or asking whether to sign off must not let
the sentence fall at the end of its reply, where it will be read as the thing itself. Refer to it, or quote
it inline in backticks mid-paragraph, but never as a closing line.

### If the session resumes afterwards

A signed-off session that gets another prompt is a new arrival: register again, re-arm the watch, and say
so in the log. The terminal block said your silence was deliberate, so a peer is entitled to act on that
until you retract it.
