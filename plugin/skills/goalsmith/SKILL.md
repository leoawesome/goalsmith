---
name: goalsmith
description: Turn a one-line brief into a battle-tested /goal completion contract for an unattended multi-hour Claude Code run. Researches the subject, drafts requirements, interviews the user until every pre-run unknown is closed, writes GOAL.md + PLAN.md + verify.sh + JOURNAL.md, stress-tests them with adversarial challengers, then hands off a paste-ready condition. Use when the user says "goalsmith", "write me a /goal prompt", "set up a goal run", "prep a long autonomous run", or hands over a brief like "clone survivor.io" expecting an autonomous build.
---

# Goalsmith

Produce the completion contract for a `/goal` run that must survive hours of unattended work without stopping, drifting, or faking success.

You are not doing the work. You are arming the loop that does it.

## Why this is hard

`/goal` is a Stop hook whose judge is a small fast model with **no tool access** — it cannot run a command or open a file, only read what the agent has already printed into the conversation. The condition is the directive that starts the first turn, and it may be up to 4,000 characters. It cannot infer intent. Every hour of unattended runtime is an hour where nobody catches a bad assumption. Two failure modes dominate:

1. **The loop stops early** — vague condition, judge says done at turn 3, you get a stub.
2. **The loop never stops** — unfalsifiable condition, or the run hits an unanswerable question at hour two and spins.

A third, worse one: **the loop declares success it did not earn** — weakens a test, adds `|| true`, marks a phase done with no evidence.

Everything below exists to close those three.

Read `references/goal-mechanics.md` before writing any condition. It documents the real evaluator contract, verified against the shipped binary — not the marketing description.

## Hard gates

<HARD-GATE>
Never print the handoff until all four are true:
1. Every item on the Pre-Run Unknowns checklist is resolved — answered by the user, answered by research, or recorded as an explicit assumption in GOAL.md.
2. `verify.sh` exists and you have RUN it. It must exit non-zero right now (nothing is built yet). A gate that passes before work starts is not a gate.
3. All five challengers returned `NO_ISSUES`, or their findings were incorporated and the re-run came back clean.
4. The condition is ONE paste that opens with the directive and stays under 4,000 characters, measured — not estimated. There is no separate kickoff line; `/goal` starts a turn with the condition itself as the directive.
5. Every clause is demonstrable from what the agent PRINTS. The judge has no tools — a clause naming a command does not make the judge run it.
</HARD-GATE>

<HARD-GATE>
Do not write feature code, scaffold the project, or start the work. You produce the contract and stop. Phase P0 of the plan does the scaffolding, during the run.
</HARD-GATE>

Do not create the git branch either. The contract instructs the run to create it. You only generate files under `.claude/goal/<slug>/`.

## Process

Create a task per item and work them in order.

1. **Frame** — classify the job, pick the slug
2. **Research** — parallel fan-out, write `BRIEF.md`
3. **Draft requirements** — from BRIEF, before asking anything
4. **Interview** — one question at a time, until no decision-changing unknown remains
5. **Generate** — `GOAL.md`, `PLAN.md`, `verify.sh`, `JOURNAL.md`
6. **Challenge** — 5 adversarial agents, revise, re-run, cap 3 rounds
7. **Hand off** — print the single paste for review, copy it to the clipboard, demand a fresh session

### 1. Frame

Classify the job. The class picks the verification pattern — see `references/verification-ladder.md`.

`greenfield` · `feature` · `migration` · `refactor` · `bug-hunt` · `debt-burndown` · `research-artifact` · `infra`

If the brief spans several classes, name the dominant one and note the others; the plan gets phases from each.

Slug: short, kebab, from the subject (`survivor-clone`, `stripe-to-adyen`, `auth-refactor`). All artifacts go in `.claude/goal/<slug>/` relative to the working directory. Create it now.

### 2. Research

Fan out in **one** message, parallel, capped at 5 agents. Never run these serially — and never dump raw findings into your own context; each agent returns a synthesis, not a page.

Thread menu — pick what the brief actually needs:

- **Subject** — what is the named thing? For a clone or port: core loop, systems, progression, session length, the 20% of features that carry 80% of the identity. For a library or service: API surface, semantics, gotchas.
- **Repo** — does code exist here? Stack, package manager, test runner, existing gates (`npm test`, CI config, lint), conventions from CLAUDE.md/AGENTS.md, and what is already built. Skip for true greenfield in an empty dir.
- **Prior art** — existing implementations, reference repos, tutorials, libraries that remove whole phases. A found engine or SDK can delete a day of the plan.
- **Verification surface** — what can actually be measured in this domain? Headless harnesses, golden-file corpora, benchmark tools, schema validators. This thread is the one people skip and it is the one that decides whether the run can be gated at all.
- **Hazards** — licensing, paid APIs, rate limits, credential needs, platform quirks, anything requiring a human hand mid-run.

Write `BRIEF.md`: findings with sources, then a **Gaps** section — every question research could not close. The Gaps section is the interview's agenda.

### 3. Draft requirements

Before asking the user anything, write down what you would build from BRIEF.md alone: the outcome, the phase skeleton, the gates you would use, the thresholds you would pick. Show it. This is what makes the interview cheap — the user reacts to a concrete proposal instead of answering an interrogation from zero.

### 4. Interview

**One question per message. Unbounded — keep going until nothing decision-changing is left.** Every question carries your recommended answer and the reason. If the answer is knowable by reading the repo or the web, go read it instead of asking.

Ask in dependency order: settle the questions that change other questions first (stack before file layout, scope before thresholds).

**Pre-Run Unknowns — every one must be closed before handoff:**

| # | Unknown | Why an unattended run dies without it |
|---|---------|----------------------------------------|
| 1 | Definition of done — the observable end state | Judge cannot rule on an unnamed target |
| 2 | Stack locked: language, framework, package manager, versions | A stack decision at hour two forks the whole build |
| 3 | Scope boundaries — paths writable, paths forbidden | Run wanders into unrelated code, diff becomes unreviewable |
| 4 | Invariants — behavior, APIs, data that must not change | Silent regression, discovered days later |
| 5 | Verification surface — gate commands that exist vs must be built | No gate means no loop, only drift |
| 6 | Thresholds as **bands**, not points — perf, coverage, size, latency | Point targets get chased forever or gamed |
| 7 | External deps — credentials, paid APIs, network, assets, licenses | Every one is a potential mid-run halt |
| 8 | Decision authority — what the run may decide alone | Run stalls on a call it was never allowed to make |
| 9 | Fixture and asset provenance | "Add test data" becomes an unanswerable question |
| 10 | Forbidden actions — deploy, push, publish, spend, touch prod | Unattended runs do exactly what you failed to forbid |
| 11 | Deliverable location and format | Work lands somewhere you never look |
| 12 | Runtime and budget tolerance | Sizing the plan and the strike limits |

Item 8 deserves its own beat. Ask directly: *"During the run, what is it allowed to decide without you?"* Then write the answer into GOAL.md as a decision-authority section, because **the run cannot ask you anything**. Anything not pre-authorized becomes either a recorded assumption or a stall.

Stop interviewing when the remaining unknowns would not change the contract. Say so and move on.

### 5. Generate

Write the artifact set using `references/templates.md`: `baseline.json`, `verify.sh`, `gates.sha256`, `PLAN.md`, `GOAL.md`, `JOURNAL.md`, `condition.txt`. Order matters — `verify.sh` before `PLAN.md`, because knowing how a phase is proven changes how it is cut.

Two properties of `PLAN.md` do the heavy lifting and are worth real effort:

**Cut phases for parallelism.** Every phase carries `Depends` and `Writes`. Phases sharing a `Depends` set with **disjoint** `Writes` paths run concurrently as subagents; overlapping paths force sequential execution. So the way you decompose decides whether the run takes three hours or nine. Deliberately split along file boundaries — per-module, per-endpoint, per-system — rather than along conceptual ones that all touch the same files. Where separation is genuinely impossible, say so in the phase block so the run does not have to work it out.

**Make it resumable.** Markers plus one commit per phase are what let a killed run restart from a single paste. Commits are the source of truth; markers are a cache of them. Any phase whose completion is not provable from `git log` is a phase the run cannot safely resume, which usually means it is too big.

Then **run `bash .claude/goal/<slug>/verify.sh`**. It must fail, and its output must name which gate failed and why. A gate whose failure message is unreadable is a gate that teaches the run nothing.

Compose the condition per `references/goal-mechanics.md`. Measure it:

```bash
printf '%s' "<condition>" | wc -c   # must be ≤ 4000
```

### 6. Challenge

Dispatch all five challengers from `references/challengers.md` in **one** message, parallel. Each gets the brief, the interview outcomes, and the full artifact set.

- All `NO_ISSUES` → proceed, say nothing about it.
- Findings → incorporate the valid ones, re-dispatch. Cap 3 rounds. Report only what changed.
- A challenger claiming a broken gate must include the exploit. No exploit, no finding.

Reward-Hacker is the one that matters most. If it finds a way to green the gate without doing the work, the contract is not shippable — fix the gate, not the wording.

### 7. Hand off

<HARD-GATE>
The handoff is **ONE paste**. `/goal` starts a turn immediately with the condition as the
directive, so a separate kickoff line is not just redundant — it wastes the first turn and
invites the user to send it in the wrong order. Never print two steps. Never print a "then
send this" line. The condition opens with the directive and carries the completion clauses
after it.
</HARD-GATE>

Print, in this order:

1. **The single paste**, verbatim, in one code block, beginning `/goal `, with its measured
   character count. Nothing follows it — no kickoff, no step 2.
2. **What the run will do** — the phase list, one line each, with the gate that closes it.
3. **What you assumed** — every recorded assumption, so a wrong one is caught now, not at hour
   four.
4. **What the judge will actually see** — name the evidence the agent must print every turn.
   The most common contract defect is a clause only a tool-using judge could check.

Write all of it to `HANDOFF.md`. Copy the condition to the clipboard:

```bash
pbcopy < .claude/goal/<slug>/condition.txt
```

Then state plainly: **run this in a fresh session.** This session is carrying research, an
interview and five challenger reports — all dead weight the run would drag through every
compaction. The contract lives on disk precisely so the run can start at near-zero context.

Offer the background invocation for a day-long run:

```bash
claude --bg --dangerously-skip-permissions
```

and the headless form, which needs no second message either:

```bash
claude -p "$(cat .claude/goal/<slug>/condition.txt)" --output-format stream-json --verbose
```

## Anti-patterns

**"The gate can be added later."** Then there is no loop, only an agent with an opinion about its own work.

**Subjective words in the condition.** "Clean", "polished", "production-ready", "good". A mechanical judge cannot rule on taste. Convert to a measurable proxy or move it out of the condition and into a human-review artifact.

**A single monolithic phase.** One phase means one gate, at the end. The run gets no signal for hours, no rollback point, and nothing to resume from.

**A sequential plan for independent work.** Phases that share no files should run concurrently. A plan whose `Writes` sets all overlap has serialised itself by accident.

**Point thresholds.** `60fps` invites gaming; `≥55fps under 300 entities, measured by <cmd>` does not.

**Trusting the transcript.** The judge sees only a compacted transcript and will rule "not yet met" on work it cannot see. Evidence must be re-stated in the final message every turn — on disk is for the agent's recovery and the human's audit, not for the judge.

**Writing a clause only a tool-using judge could check.** The judge has no tools. `` `bash verify.sh` exits 0 `` does not cause anything to run and cannot be verified by the judge; it is a claim it hunts for in the transcript. Require the *output*, in this turn's final message. This is the single most common way a contract silently degenerates into the agent grading itself.

**Splitting the handoff.** `/goal` starts a turn with the condition as the directive. A separate "now read GOAL.md and begin" message is redundant, wastes the first turn, and can arrive out of order. One paste, directive first.

**Letting the run infer authority.** Unattended agents resolve ambiguity by guessing. Pre-authorize, or expect a guess.
