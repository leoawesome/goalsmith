# How `/goal` actually works

Verified against the official documentation at https://code.claude.com/docs/en/goal (checked
2026-08-20). Three things in earlier versions of this file were **wrong** and had been shaping
contracts badly — they are corrected below and called out explicitly, because a contract built
on the old assumptions is subtly broken.

## The mechanism

`/goal <condition>` registers a **session-scoped prompt-based Stop hook**. One goal per
session; setting a new one replaces it.

> "Setting a goal starts a turn immediately, with the condition itself as the directive. You
> don't need to send a separate prompt."

After each turn, Claude Code sends the condition **and the conversation so far** to the
configured small fast model (Haiku by default). It returns one of three verdicts with a reason:

- **Not yet met** → Claude keeps working and takes the reason as guidance for the next turn.
- **Met** → the goal clears, an achieved entry is recorded.
- **Impossible** → the goal clears, a failed entry is recorded with the reason.

## The three corrections

### 1. The condition IS the kickoff. Never split them.

The old version of this file told you to hand over a condition *and* a separate kickoff line
("Read GOAL.md and execute it"). That is wrong, and it wastes the first turn: `/goal` starts a
turn immediately using the condition as the directive.

So the condition must **open with the directive** and then state the completion clauses:

```
Execute .claude/goal/<slug>/GOAL.md per its turn protocol. DONE when: …
```

One paste. One prompt. There is no second step.

### 2. The evaluator has NO TOOLS.

> "The evaluator runs on whichever provider your session is configured for. It does not call
> tools, so it can only judge what Claude has already surfaced in the conversation."

> "The evaluator judges your condition against what Claude has surfaced in the conversation.
> It doesn't run commands or read files independently, so write the condition as something
> Claude's own output can demonstrate."

The old version of this file claimed the opposite — that the judge could read files and run
commands, and that naming a command in the condition meant the judge would execute it. **It
will not.** A condition like `` `bash verify.sh` exits 0 `` does not cause anything to run; it
is a claim the judge looks for evidence of in the transcript.

This changes how every clause must be written. The pattern is: **name the command, and require
its output in the turn's final message.**

```
DONE when the final message pastes `bash <dir>/verify.sh` output ending in ALL GATES PASS,
with the HEAD sha it ran at.
```

The gate script still matters enormously — it is what makes the agent's claim expensive to
fake, and what a human checks afterwards. But the *judge* is reading a transcript, so the
contract has to make the agent print the proof.

Corollary: a gate that prints a single terminal line (`ALL GATES PASS`, `ALL GATES FAILED`) is
worth far more than one that prints only an exit code, because the line is what lands in the
transcript.

### 3. The cap is 4,000 characters, not 500.

> "The condition can be up to 4,000 characters."

The old 500-char figure caused contracts to be compressed into near-unreadable shorthand for no
reason. 4,000 characters is enough for the directive, numbered clauses, the negative clauses,
the evidence requirement, and the impossible authorization — written as sentences.

Still don't pad it. Every clause has to be checkable by a model reading a transcript with no
tools. Length buys clarity, not more conditions.

## Six more consequences

### The transcript is the only evidence, and it gets long

The judge sees the conversation so far. On a multi-hour run the early turns are compacted away.
Therefore:

- Evidence must be **re-stated in the final message of every turn**, not left in an earlier one.
- `JOURNAL.md` on disk is for the *agent's* recovery and for the human's audit — the judge
  cannot read it. Anything the judge must see has to be printed.
- A silent turn is an invisible turn.

### The `reason` string is the run's steering

`Not yet met` feeds the reason back as guidance for the next turn. Write the condition so its
rejection is instructive: numbered clauses yield "clause 3 failed", one long sentence yields
"not done yet", which steers nothing.

### `impossible` is the clean exit, and the agent must earn it

The agent cannot set the verdict. It can only surface a blocker clearly enough that the judge
rules impossible. So the condition must **authorize** it with a precondition strict enough that
a lazy run can't trigger it at hour one:

```
Judge impossible only if the final message states BLOCKED and names 3 failed attempts plus
1 documented alternate approach on the same phase.
```

Without such a clause there is no clean exit. With a loose one the run bails on first contact.

Note the blocker has to be **in the message**, not only in a file.

### The no-progress guard will stop the loop

> "If Claude keeps answering the evaluator without making progress (no tool use for several
> turns in a row), Claude Code stops the loop, prints a warning, and returns control to you
> with the goal still set."

A run that argues with the judge instead of working gets halted. This is a safety net, not a
budget — it does not bound a run that *is* using tools.

To bound length, put it in the condition: `or stop after 20 turns`. Claude reports progress
against that clause and the judge reads it from the conversation.

### Background work defers evaluation

If a subagent or background shell is still running when a turn ends, evaluation is **skipped**
for that turn and happens at the end of the next turn with nothing running. When background
work finishes, the result is delivered as a new turn.

Consequence for parallel plans: a fan-out turn is not judged, so it cannot be failed for
missing evidence. Good. But after 30 minutes of waiting, Claude Code asks the agent to check on
the background work (tunable via `CLAUDE_CODE_GOAL_CHECKIN_MINUTES`, `0` disables).

### Four failures clear the goal; the rest don't

Auth failure (when Claude Code manages its own credentials), exhausted credit balance, a
context overflow auto-compaction couldn't clear, and an unavailable model. The warning starts
`Goal cleared after an unrecoverable error`. Everything else — rate limits, overloaded servers
— leaves the goal active.

## Environment requirements

- **Trusted workspace only** — `/goal` is part of the hooks system.
- Unavailable when `disableAllHooks` is true, or `allowManagedHooksOnly` is set in managed
  settings. It tells you why rather than silently doing nothing.
- **Permission mode is unchanged by `/goal`.** For unattended turns, run it in auto mode or
  with permissions pre-granted; otherwise Claude still asks before unapproved tool calls.
- **Resume**: a goal active when the session ended is restored by `--resume`/`--continue`. The
  condition carries over; turn count, timer and spend baseline reset. An achieved or cleared
  goal is not restored.
- **Clear** with `/goal clear` (aliases: `stop`, `off`, `reset`, `none`, `cancel`). `/clear`
  also removes it. `/goal` with no argument shows status.
- **Headless**: `claude -p "/goal <condition>"` runs the loop to completion in one invocation.
  Add `--output-format stream-json --verbose` or nothing prints until the end.

## Condition template

One paste. Directive first, then clauses.

```
Execute .claude/goal/<slug>/GOAL.md per its turn protocol.

DONE when your final message pastes, in this turn: (1) `bash .claude/goal/<slug>/verify.sh`
output whose last line is ALL GATES PASS; (2) the HEAD sha it ran at; (3) `git status
--porcelain` showing a clean tree.

Not done if that output is absent, is from an earlier turn, or shows any FAIL line. Every turn
must end by restating which phases completed, the gate result, and the next action.

Judge impossible only if the final message says BLOCKED and names 3 failed attempts plus 1
alternate approach on one phase.
```

Anatomy — five parts, all required:

1. **The directive** — what to do, first, because the condition starts the turn.
2. **The evidence requirement** — what must appear *in the message*. This replaces the old
   "mechanical gate" idea: the gate runs, but the judge reads the paste.
3. **Freshness** — "in this turn", so a stale paste from turn 3 doesn't satisfy turn 40.
4. **Negative clauses** — the fake-done states you predict.
5. **Impossible authorization** — the clean exit, strictly gated.

The condition never changes across restarts. That is what makes a killed run recoverable from a
single paste.

## Testing a condition before you ship it

Read it back as a model with **no tools**, seeing only a conversation:

- **Could I return `met` right now, with nothing done?** If yes, it's empty.
- **Is there a state where the work is genuinely finished and I still say no?** Especially:
  does any clause require something I cannot see, like a file's contents or an exit code nobody
  printed?
- **If I say no, does my reason tell the run what to do next?**
- **Can I tell when I'm allowed to say impossible?**

Then hand the same questions to the Evaluator-Sim challenger, and tell it the judge has no
tools — it is the single most common thing to get wrong.
