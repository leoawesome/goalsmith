# Artifact templates

Everything lives in `.claude/goal/<slug>/`. Write in this order: `BRIEF.md` → `baseline.json` → `verify.sh` → `PLAN.md` → `GOAL.md` → `JOURNAL.md` → `condition.txt` → `HANDOFF.md`.

`verify.sh` comes before `PLAN.md` deliberately: knowing how a phase is proven changes where you cut phases.

Replace every `<...>`. Delete sections that genuinely do not apply — an empty section is noise the run has to read every turn.

---

## GOAL.md

The contract. The run re-reads this at the start of every turn, so it must stay short enough to survive that — target under 200 lines. Detail belongs in PLAN.md.

```markdown
# Goal: <one sentence, observable end state>

Job class: <greenfield|feature|migration|refactor|bug-hunt|debt-burndown|research-artifact|infra>
Directory: .claude/goal/<slug>/
Armed: <YYYY-MM-DD>

## Outcome

<2-4 sentences. What exists when this is done, stated so someone who never
read the brief could recognise it. No adjectives that cannot be measured.>

## Verification

Single gate: `bash .claude/goal/<slug>/verify.sh` — exit 0 means done.

| Layer | Checks | Command |
|-------|--------|---------|
| L0 integrity | <gate files unmodified, tree clean, no escape hatches> | <cmd> |
| L1 build | <> | <cmd> |
| L2 static | <> | <cmd> |
| L3 unit | <> | <cmd> |
| L4 integration | <> | <cmd> |
| L5 thresholds | <bands + monotone counters vs baseline.json> | <cmd> |
| L6 artifacts | <required deliverables exist, non-trivial> | <cmd> |

Thresholds are bands, both edges named, in baseline.json.

## Boundaries

Writable: <paths>
Forbidden: <paths — dependencies, generated dirs, unrelated modules, other people's code>

Widening this list is not permitted. If the work appears to require a forbidden
path, that is a stop_when condition, not a judgement call.

## Invariants — must not change

- <behavior / public API / schema / data / config that must survive>
- <...>

## Decision authority

Decide alone, record in JOURNAL, move on:
- <e.g. library choice within the locked stack; file layout; naming; test structure>

Never decide alone — these are stop_when:
- <e.g. anything costing money; schema changes; auth model; scope additions>

## Forbidden actions

- Never push, deploy, publish, or release.
- Never spend money or hit paid APIs beyond <named allowance, or "none">.
- Never touch production or shared state.
- Never weaken, skip, delete, or bypass a test or gate to make verify pass.
- Never modify verify.sh, baseline.json, or gates.sha256. Adding new tests is
  allowed and expected; changing what counts as passing is not.
- Never mark a phase [x] without pasting the gate output into JOURNAL.md.
- Never expand Boundaries.
- <domain-specific additions>

## Turn protocol

Every turn, in order:

1. **Re-orient from disk.** Read PLAN.md and the last 3 JOURNAL.md entries
   before acting. Do not rely on conversation memory — this session will be
   compacted repeatedly and earlier turns will be gone.
2. **Compute the ready set** — every phase not `[x]` whose `Depends` are all
   `[x]`. Treat any `[~]` as `[ ]` (see Resume protocol).
3. **Dispatch.**
   - Ready set of 1, or overlapping `Writes` paths → do it yourself, in this turn.
   - Ready set of 2+ with **disjoint** `Writes` paths → fan out, one subagent per
     phase, all in a single message, max <N> at a time. See Parallelism.
4. **Delegate wide work.** Exploration, multi-file reads, and verification runs
   go to subagents; take back the conclusion, not the transcript. Raw file dumps
   and full logs must never enter the main context.
5. **Run each phase gate.** Capture its output verbatim.
6. **Commit** — you, never a subagent. One commit per phase, sequentially:
   `<type>(<slug>): <phase> — <what>`.
7. **Mark and journal.** Set each phase `[x]`, refresh the PLAN.md Status line,
   append one JOURNAL entry per phase, and **end the turn by restating**: phases
   completed, gate results, HEAD sha, next action. The evaluator sees a truncated
   transcript — an unstated result is an invisible result.

## Parallelism

Independent work runs in parallel. Sequential-by-default wastes hours on a
multi-phase build.

Two phases may run concurrently only when **all** hold:

- Both are in the ready set — every `Depends` already `[x]`.
- Their `Writes` path sets are disjoint. Any overlap, even one shared file, means
  sequential. Shared config, lockfiles, and barrel/index files count as overlap.
- Neither modifies dependencies, migrations, or generated output the other reads.

Rules for the fan-out:

- Max <N> concurrent subagents. All dispatched in one message, never serially.
- Each subagent gets: its phase block from PLAN.md, GOAL.md Boundaries and
  Invariants, its `Writes` allowlist, and its gate command.
- **Subagents never run git write commands.** No commit, no add, no branch, no
  stash. They edit files and run their gate; you own the index. Parallel commits
  race and corrupt the one-commit-per-phase audit trail.
- Each subagent returns: files changed, gate command, verbatim gate output,
  PASS or FAIL, and any assumption it recorded. Not a transcript.
- Any subagent returning FAIL → that phase stays `[ ]` and its attempt counter
  increments. Its siblings' passes still commit. Do not roll back good work.
- After the fan-out, run the phase gates once more yourself before committing.
  A subagent reporting PASS is a claim; the gate is the fact.

When in doubt about disjointness, go sequential. A lost hour beats a corrupted
tree with no clean rollback point.

## Resume protocol

This run may be killed mid-flight — usage limits, provider outage, a closed
laptop. Restarting must be one paste, not an investigation.

Restarting is the identical handoff: the same single `/goal` paste, which is both the
condition and the directive. The contract is on disk and idempotent. Before doing any new work on the first
turn after a restart:

1. Read PLAN.md Status, then the last 5 JOURNAL entries.
2. `git log --oneline <base_sha>..HEAD` — what actually landed. **Commits are the
   truth.** A phase marked `[x]` with no commit behind it was interrupted; reset
   it to `[ ]`.
3. `git status --porcelain` — uncommitted work from the killed turn. Either
   finish it and commit it under its phase, or discard it. Never leave it to be
   half-adopted by a later phase.
4. Any `[~]` phase: treat as `[ ]`. Keep its attempt counter from JOURNAL — an
   interrupted attempt is not a failed attempt, so do not increment it.
5. Append a JOURNAL entry `RESUME` naming the recovered phase, HEAD sha, and what
   was discarded. Then continue the normal turn protocol.

Never restart by re-running finished phases, and never assume an unmarked phase
is unstarted — check the commits.

## Stall policy

Per phase:
1. Attempt fails → log root cause in JOURNAL. Retry with a fix. (attempt 2)
2. Fails again → log. Retry. (attempt 3)
3. Third failure → log, then try **one materially different approach**. A
   different library, a different decomposition, a different algorithm. Another
   retry of the same approach does not count and does not reset the counter.
4. Alternate approach also fails → state plainly in the final message that the
   goal is blocked, name the blocker, cite the JOURNAL entries. The evaluator
   will judge it impossible and the loop exits cleanly.

Never silently skip a phase. Never mark a blocked phase `[x]`.

## stop_when — earn the impossible verdict

- Three failed attempts plus one alternate approach on a single phase.
- A required credential, asset, license, or paid resource is unavailable.
- The work would require touching a Forbidden path or breaking an Invariant.
- A decision outside Decision authority blocks all remaining phases.

In every case: state the blocker plainly in the final message. Do not stop
working silently — a silent stop reads to the judge as an unfinished turn.

## Assumptions recorded at arming time

- <every gap closed by assumption rather than by an answer>
- <...>

## Cannot ask

There is no human in this loop. Any question you would ask must be resolved by
Decision authority, by an existing repo convention, or by recording an
assumption in JOURNAL.md and proceeding. Only a stop_when blocker justifies
halting.
```

---

## PLAN.md

Phases in dependency order. Each has one gate and one commit. Small enough that a phase fits in a turn or two — a phase spanning many turns loses its rollback point and its signal.

```markdown
# Plan: <slug>

## Status

Derived — regenerate every turn. If this disagrees with the phase markers below,
**the markers win**.

Done: <none> · Active: <none> · Blocked: <none> · HEAD: <sha> · Updated: <timestamp>

## Markers

`[ ]` todo · `[~]` claimed this turn · `[x]` done, gate passed, output in JOURNAL · `[!]` blocked

`[~]` is a crash marker, not a state: on resume it reverts to `[ ]`.
`[!]` is not a finish state — valid only alongside a stop_when JOURNAL entry.
`[x]` requires a commit. A `[x]` with no commit behind it was interrupted.

## P0 — <harness / baseline>
- [ ] <deliverable>
- Gate: `<command>` — <what passing means>
- Depends: none
- Writes: <paths this phase may modify>
- Est: <turns>

## P1 — <first system>
- [ ] <deliverable>
- [ ] <tests landing with it>
- Gate: `<command>`
- Depends: P0
- Writes: <paths — disjoint from siblings that share a Depends set>
- Est: <turns>

## P2 — <second system, parallel-safe with P1>
- [ ] <deliverable>
- Gate: `<command>`
- Depends: P0
- Writes: <paths — must not intersect P1's>
- Est: <turns>

<... one block per phase ...>

## PF — integration
- [ ] full `verify.sh` green on a clean tree
- [ ] artifacts written to .claude/goal/<slug>/artifacts/
- [ ] final JOURNAL entry: HEAD sha + full verify output
- Gate: `bash .claude/goal/<slug>/verify.sh`
- Depends: <all>
- Writes: <paths>
```

For greenfield, P0 is always the harness. Never features before the gate that proves them.

`Depends` and `Writes` are what make parallelism safe — they are the dependency
graph and the lock table. Cut phases so that siblings sharing a `Depends` set have
disjoint `Writes`; that is what turns a sequential plan into a parallel one. Where
you cannot separate them, say so in the phase block so the run does not have to
guess.

---

## verify.sh

```bash
#!/usr/bin/env bash
# Single gate for goal <slug>. Exit 0 = done. Any other code = not done.
# IMMUTABLE during the run. Adding tests is allowed; changing pass criteria is not.
set -uo pipefail

DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT="$(git -C "$DIR" rev-parse --show-toplevel 2>/dev/null || echo "$DIR")"
cd "$ROOT" || exit 3
FAILED=0

fail() { printf 'FAIL [%s] %s\n' "$1" "$2" >&2; FAILED=1; }
pass() { printf 'ok   [%s] %s\n' "$1" "$2"; }

# ---- L0 integrity -------------------------------------------------------
# Runs first: if the gate was tampered with, nothing below is trustworthy.
if [ -f "$DIR/gates.sha256" ]; then
  if sha256sum -c "$DIR/gates.sha256" >/dev/null 2>&1; then
    pass L0 "gate files unmodified"
  else
    fail L0 "verification files were MODIFIED — this invalidates the run"
    exit 1
  fi
fi

[ -z "$(git status --porcelain 2>/dev/null)" ] \
  && pass L0 "tree clean" || fail L0 "uncommitted changes — commit each phase"

BASE_SHA=$(jq -r '.base_sha // empty' "$DIR/baseline.json" 2>/dev/null)
if [ -n "$BASE_SHA" ]; then
  if git diff "$BASE_SHA"..HEAD -- <TEST_PATHS> 2>/dev/null | \
     grep -qE '^\+.*(\.skip|\.only|xit\(|@Ignore|pytest\.mark\.skip|# type: ?ignore|@ts-ignore|\|\| true|--no-verify|continue-on-error)'; then
    fail L0 "test escape hatch introduced — see git diff on test paths"
  else
    pass L0 "no escape hatches"
  fi

  BASE_TESTS=$(jq -r '.test_count // 0' "$DIR/baseline.json")
  NOW_TESTS=$(<COUNT_TESTS_CMD>)
  [ "$NOW_TESTS" -ge "$BASE_TESTS" ] \
    && pass L0 "test count $NOW_TESTS >= $BASE_TESTS" \
    || fail L0 "tests were deleted: $BASE_TESTS -> $NOW_TESTS"
fi

# ---- L1 build -----------------------------------------------------------
<BUILD_CMD> >/dev/null 2>&1 && pass L1 "build" || fail L1 "build failed: run <BUILD_CMD>"

# ---- L2 static ----------------------------------------------------------
<LINT_CMD> >/dev/null 2>&1 && pass L2 "lint" || fail L2 "lint failed: run <LINT_CMD>"

if grep -rniE '\b(TODO|TBD|FIXME|lorem ipsum|placeholder)\b' <SRC_PATHS> >/dev/null 2>&1; then
  fail L2 "placeholder markers left in source"
else
  pass L2 "no placeholders"
fi

# ---- L3 unit ------------------------------------------------------------
<TEST_CMD> >/dev/null 2>&1 && pass L3 "unit tests" || fail L3 "unit tests failed: run <TEST_CMD>"

# ---- L4 integration ----------------------------------------------------
<INTEGRATION_CMD> >/dev/null 2>&1 && pass L4 "integration" || fail L4 "integration failed: run <INTEGRATION_CMD>"

# ---- L5 thresholds -----------------------------------------------------
# Bands from baseline.json. Both edges. Named measurement command each.
# <METRIC>: measure, compare to [lo, hi], fail with the actual number.

# ---- L6 artifacts ------------------------------------------------------
for a in <REQUIRED_ARTIFACTS>; do
  if [ -s "$DIR/artifacts/$a" ]; then pass L6 "artifact $a"
  else fail L6 "missing or empty artifact: $a"; fi
done

# ------------------------------------------------------------------------
if [ "$FAILED" -eq 0 ]; then
  printf '\nALL GATES PASS\n'; exit 0
else
  printf '\nGATES FAILED — goal not met\n'; exit 1
fi
```

Two properties matter more than coverage: every failure message **names the command to reproduce it**, and integrity runs first. That failure text becomes the judge's `reason`, which becomes the run's next instruction.

After writing it:

```bash
chmod +x .claude/goal/<slug>/verify.sh
sha256sum .claude/goal/<slug>/verify.sh .claude/goal/<slug>/baseline.json \
  > .claude/goal/<slug>/gates.sha256
bash .claude/goal/<slug>/verify.sh   # MUST fail, for a readable reason
```

---

## baseline.json

State recorded at arming time. Without it, monotone counters and parity checks have nothing to compare against.

```json
{
  "armed": "<YYYY-MM-DD>",
  "base_sha": "<git rev-parse HEAD, or null for empty repo>",
  "test_count": <n>,
  "coverage_pct": <n or null>,
  "lint_errors": <n>,
  "type_errors": <n>,
  "corpus_size": <n or null>,
  "bands": {
    "<metric>": { "lo": <n>, "hi": <n>, "unit": "<unit>", "measure": "<command>" }
  }
}
```

---

## JOURNAL.md

The run's durable memory. It exists because the evaluator sees a truncated transcript and the run gets compacted — this file is what both of them fall back on.

```markdown
# Journal: <slug>

Append only. Newest last. One entry per turn, no exceptions.

---
## <YYYY-MM-DD HH:MM> · P<n> · <attempt k> · <PASS|FAIL|BLOCKED|RESUME>

**Did:** <one or two sentences>
**Gate:** `<command>`
```
<verbatim gate output — trimmed to the decisive lines>
```
**Commit:** <sha or "none — gate failed">
**Assumption:** <only if one was recorded this turn>
**Next:** <the single next action>
---
```

One entry per phase, not per turn — a parallel turn that closed P1 and P2 writes two entries.

A `RESUME` entry replaces **Gate** with:

```
**Recovered:** <phase reset from [~] to [ ]>
**Landed:** <commits found at <base>..HEAD>
**Discarded:** <uncommitted work thrown away, or "none">
```

Rules the run must follow:

- Append for every phase touched, including failed and blocked ones. A missing entry is indistinguishable from a lost turn.
- `PASS` requires real gate output pasted in. A `PASS` with no output is a contract violation.
- Attempt counter resets only when the phase passes. A retry does not reset it; an interrupted attempt does not increment it.
- Never rewrite or delete history. Corrections go in a new entry.

---

## condition.txt

The single paste, under 4,000 characters, nothing else in the file — it goes straight to the
clipboard. It opens with the **directive**, because `/goal` starts a turn immediately with the
condition as the directive; there is no separate kickoff message.

Remember the judge has **no tools**. Every clause must be satisfiable by what the agent PRINTS,
so require the gate output in the turn's final message rather than merely naming the command.

```
Execute .claude/goal/<slug>/GOAL.md per its turn protocol.

DONE when your final message, in this turn, pastes: (1) the output of `bash .claude/goal/<slug>/verify.sh` with ALL GATES PASS as its last line; (2) the HEAD sha it ran at; (3) `git status --porcelain` showing a clean tree.

Not done if that paste is missing, is copied from an earlier turn, or contains any FAIL line. End every turn by restating which phases closed, the gate result, and the next action.

Judge impossible only if the final message begins BLOCKED and names 3 failed attempts plus 1 alternate approach on a single phase.
```

434 characters at a 15-character slug — headroom for a longer slug or one extra
negative clause, not for prose.

```bash
printf '%s' "$(cat .claude/goal/<slug>/condition.txt)" | wc -c   # ≤ 4000
```

---

## HANDOFF.md

What the user reads before firing.

```markdown
# Handoff: <slug>

## Fire it — one paste, <n> chars

/goal <condition, opening with the directive>

(already on your clipboard — there is no second step; `/goal` starts the first turn itself)

## Run in a fresh session

This contract lives on disk so the run can start near-zero context. Firing it
from the session that authored it drags the research, interview, and challenger
reports through every compaction.

For a day-long run: `claude --bg --dangerously-skip-permissions`

## If it dies — restart is the same two steps

Usage limit, provider outage, closed laptop: open a fresh session and paste the
same single condition. Nothing to reconstruct. (A goal still active when the
session ended is also restored by `--resume`/`--continue`.)

The first turn after a restart runs the Resume protocol in GOAL.md: reconciles
PLAN.md markers against the actual commits, discards or adopts the interrupted
turn's uncommitted work, logs a RESUME entry, and continues. Finished phases are
not redone.

Check state at any time without starting anything:

```bash
sed -n '1,12p' .claude/goal/<slug>/PLAN.md     # status + markers
tail -40 .claude/goal/<slug>/JOURNAL.md        # last turns
git log --oneline <base_sha>..HEAD             # what actually landed
```

## What it will do

| Phase | Deliverable | Gate | Est |
|-------|-------------|------|-----|
| P0 | <> | <> | <> |

Estimated total: <n> turns / <n> hours.

## Assumptions — check these now

- <assumption> → wrong? fix GOAL.md before firing.

## Blockers it will stop on

- <stop_when items>

## Review after it exits

- `.claude/goal/<slug>/artifacts/` — screenshots, recordings, samples
- `.claude/goal/<slug>/JOURNAL.md` — full turn history
- `git log <base_sha>..HEAD` — one commit per phase
```
