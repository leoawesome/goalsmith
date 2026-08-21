# Verification ladder

How to build a gate for any job, and how to keep it un-gameable. Domain-agnostic: the metric *types* are the reusable part, the job classes just tell you which types apply.

## Rule zero

**The gate must be creatable before the work exists.** If you cannot write a failing check today, the run has no signal and `/goal` degenerates into an agent grading itself.

For greenfield, that means phase P0 builds the harness — not the feature. First commit is a test runner and one failing assertion.

## The six metric types

Every gate is built from these. A contract needs at least a Binary Gate and an Anti-Hack Invariant; most need three or four types.

### 1. Binary gate — exit code

One command, exit 0 or not. `npm test`, `cargo build`, `pytest`, `tsc --noEmit`.

The load-bearing metric. Non-negotiable in every contract.

### 2. Monotone counter — must move one way, never back

Record a baseline **now**, before the run, into the artifact directory. The gate compares against it.

Coverage %, lint error count, type error count, failing test count, bundle bytes, TODO count, `any` count.

```bash
BASE=$(cat "$DIR/baseline.json" | jq -r .coverage)
NOW=$(...)
awk "BEGIN{exit !($NOW >= $BASE)}" || fail "coverage regressed: $BASE -> $NOW"
```

Monotone counters are what stop a run from trading one problem for another.

### 3. Threshold band — value must land inside a range

Never a point. Bands with both edges.

`p95 latency in [0, 250]ms` · `fps ≥ 55 under 300 entities` · `time-to-kill in [1.8, 2.2]s at wave 5` · `bundle in [0, 500]kb`

A point target invites infinite chasing or a hardcoded return. A band with an upper *and* lower edge also catches degenerate solutions — a wave that is trivially survivable fails the lower edge just as a brutal one fails the upper.

Every band needs a named measurement command. A band nobody can measure is a wish.

### 4. Corpus parity — old vs new over N cases

For migrations and ports. Freeze a corpus, record expected outputs, assert equality or tolerance.

```
for each of N fixtures: new_impl(input) == golden(input)   # N recorded in baseline
```

Corpus size is itself a gate: `N ≥ <count>`. Otherwise the run shrinks the corpus until parity is trivial.

### 5. Existence and shape

The deliverable exists, parses, and is non-trivial. Guards against the empty-file finish.

File exists · matches schema · ≥ N sections · ≥ N entries · every claim has a source line · no placeholder markers (`TODO`, `TBD`, `lorem`, `FIXME`, `...`).

Minimum-size checks matter more than they look. `wc -l` floors and required-section greps are what stop a two-line "report".

### 6. Anti-hack invariants — the gate guards itself

The one people forget, and the one Reward-Hacker will attack first. An unattended run under pressure will absolutely weaken its own gate.

```bash
# gate files unchanged since arming
sha256sum -c "$DIR/gates.sha256" || fail "verification files were modified"

# test count never shrinks
[ "$(count_tests)" -ge "$BASELINE_TESTS" ] || fail "tests were deleted"

# no escape hatches introduced
! git diff "$BASE_SHA"..HEAD -- <test paths> | grep -E '^\+.*(\.skip|\.only|xit\(|@Ignore|pytest.mark.skip|# type: ignore|@ts-ignore|\|\| true|--no-verify|continue-on-error)' \
  || fail "test escape hatch added"

# clean tree — no uncommitted work masquerading as done
[ -z "$(git status --porcelain)" ] || fail "uncommitted changes"
```

Checksum the gate files at arming time and write `gates.sha256` next to them. If the run needs to *extend* the gate (normal — greenfield adds tests as it goes), allow appends to test files but keep `verify.sh` itself and any threshold config immutable. State that split explicitly in GOAL.md.

## Job classes

### greenfield

Nothing exists, so nothing can be measured. Sequence is forced:

- **P0** — scaffold, dependency install, test runner, CI-less `verify.sh`, one deliberately failing assertion, first commit. Gate: `verify.sh` runs and fails *for the right reason*.
- **P1..Pn** — one system per phase, each landing tests with it.
- **Final** — integration harness. For anything interactive or visual: a **seeded headless simulation** asserting numbers, plus screenshots/recordings written to an artifacts folder for human review after the run. Nothing subjective inside the gate.

Types: Binary, Band, Existence, Anti-hack.

### feature

Existing code, new behavior. Baseline the suite first — if it is red today, that is a finding, not a starting line.

Gate: full suite green + new tests covering each acceptance criterion + invariant list untouched.

Types: Binary, Monotone (coverage, don't regress), Anti-hack.

### migration / port

Parity is the whole job. Build the corpus in P0, before touching the implementation.

Gate: corpus parity at N cases + both implementations still buildable during the transition + explicit cutover phase.

Types: Corpus parity, Binary, Anti-hack.

### refactor

Behavior must not change, so the gate is invariance:

Suite green before and after · public API diff empty or matching an explicit allowlist · coverage not down · performance band unchanged.

The pre-state must be recorded in `baseline.json` at arming time. A refactor with no recorded baseline cannot be verified at all.

Types: Binary, Monotone, Corpus parity (API surface), Anti-hack.

### bug-hunt

Order is the gate: **failing repro first**, then the fix.

Each bug: a test that fails on the parent commit and passes on HEAD. That pairing is the proof; a test written after the fix proves nothing.

Gate: `N` bugs, each with a named regression test, each verified failing at its parent sha.

Types: Binary, Monotone (count of fixed bugs), Anti-hack.

### debt-burndown

Pure monotone work. Coverage up, lint down, `any` down, dead code out.

Gate: each tracked counter past its target, none of the others regressed, suite green.

Types: Monotone (several), Binary, Anti-hack.

### research-artifact

No code, so the gate is structural. Deliverable exists · required sections present · every claim carries a source · minimum source count · no placeholders · internal links resolve.

Write it as a script, not a vibe. `grep -c '^## '`, link checker, citation-per-claim ratio.

Types: Existence and shape, Monotone (source count), Anti-hack (no placeholder markers).

### infra

Gate: idempotent apply (running twice changes nothing the second time) · health check returns expected status · smoke suite green · rollback path exercised, not just written.

Types: Binary, Existence, Band (latency), Anti-hack.

## Where subjective quality goes

Out of the condition, into artifacts.

The run writes screenshots, recordings, sample outputs, and a diff summary into `<dir>/artifacts/`. The gate checks that they **exist and are non-trivial** — never that they are good. Quality review is a human pass after the loop exits.

Then convert what you can into proxies. "Feels responsive" → input-to-response latency band. "Balanced" → a seeded sim asserting win-rate inside a band. "Readable" → a complexity or line-length ceiling. Proxies are imperfect; a proxy that is measurable beats an ideal nobody can check.

## verify.sh shape

One entry point, layered, fail-fast, and **loud about which layer failed** — that message becomes the judge's `reason` and the run's next instruction.

```
L0  integrity     gate files unmodified, tree clean, no escape hatches
L1  build         compiles / typechecks / installs
L2  static        lint, format, no placeholder markers
L3  unit          test suite green
L4  integration   behavior / parity / corpus
L5  thresholds    bands and monotone counters vs baseline.json
L6  artifacts     required deliverables exist and are non-trivial
```

Integrity runs **first**. If the run tampered with its own gate, nothing downstream is trustworthy and the failure message should say exactly that.
