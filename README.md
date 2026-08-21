# Goalsmith

A Claude Code skill that turns a one-line brief into a **battle-tested `/goal` completion contract** for an unattended multi-hour run.

You don't write the prompt. You describe the outcome, answer questions, and get one paste-ready condition plus the artifact set that keeps the run honest for hours.

## The problem

`/goal` is a Stop hook. Its judge is a small, fast model with **no tool access** — it cannot run a command or open a file, only read what the agent already printed. Hand it a vague condition and one of three things happens:

1. **The loop stops early** — judge says done at turn 3, you get a stub.
2. **The loop never stops** — unfalsifiable condition, or hour two hits an unanswerable question and it spins.
3. **The loop declares success it did not earn** — weakens a test, adds `|| true`, marks a phase done with no evidence.

Goalsmith exists to close all three.

## What it produces

In `.claude/goal/<slug>/`:

| File | Role |
|---|---|
| `GOAL.md` | The contract: definition of done, scope boundaries, invariants, decision authority, forbidden actions |
| `PLAN.md` | Phases with explicit `Depends` / `Writes` so independent phases run concurrently and any phase is resumable from `git log` |
| `verify.sh` | The executable gate. Generated **before** the plan, and run to prove it fails while nothing is built |
| `baseline.json`, `gates.sha256` | Pinned starting measurements and gate integrity, so the run can't quietly relax its own thresholds |
| `JOURNAL.md` | Append-only run log |
| `condition.txt` | The single `/goal ...` paste, measured to fit under 4,000 characters |
| `HANDOFF.md` | The paste, the phase list, every recorded assumption, and exactly what the judge will see |

## How it works

1. **Frame** — classify the job (`greenfield`, `feature`, `migration`, `refactor`, `bug-hunt`, `debt-burndown`, `research-artifact`, `infra`); the class picks the verification pattern.
2. **Research** — parallel fan-out over subject, repo, prior art, verification surface, and hazards. Writes `BRIEF.md` ending in a **Gaps** section.
3. **Draft requirements** — proposes a concrete build from research alone, so the interview is a reaction, not an interrogation.
4. **Interview** — one question at a time, each carrying a recommended answer, until all **12 Pre-Run Unknowns** are closed. Including the one people forget: *what is the run allowed to decide without you?* — because an unattended run cannot ask.
5. **Generate** — the artifact set, then runs `verify.sh` to confirm it fails loudly and names which gate failed.
6. **Challenge** — 5 adversarial agents in parallel, up to 3 rounds. The one that matters is **Reward-Hacker**: if it can green the gate without doing the work, the gate gets fixed, not the wording.
7. **Hand off** — one paste, copied to your clipboard, to be run in a **fresh session**.

## Install

### As a plugin (recommended)

```
/plugin marketplace add leoawesome/goalsmith
/plugin install goalsmith
```

Then invoke with `/goalsmith:goalsmith`, or just say "goalsmith" / "write me a /goal prompt".

### As a plain skill

```bash
git clone https://github.com/leoawesome/goalsmith.git
ln -s "$PWD/goalsmith/plugin/skills/goalsmith" ~/.claude/skills/goalsmith
```

(Or `cp -R goalsmith/plugin/skills/goalsmith ~/.claude/skills/` if you'd rather not symlink.)

## Usage

```
goalsmith: clone survivor.io as a web game
```

```
goalsmith: migrate our Stripe integration to Adyen without changing the public API
```

Then answer the questions. At the end you get one line to paste into a fresh session:

```
/goal <the contract>
```

For a day-long run:

```bash
claude --bg --dangerously-skip-permissions
```

Or fully headless — no second message needed:

```bash
claude -p "$(cat .claude/goal/<slug>/condition.txt)" --output-format stream-json --verbose
```

## Requirements

- Claude Code with the `/goal` command available (Goalsmith writes the contract; `/goal` runs it).
- `git` in the working directory — phase commits are how a killed run resumes.
- The interview does real research, so web access helps but is not required.

## Design notes

Two rules do most of the work, and both are counterintuitive:

**The gate comes before the plan.** Knowing how a phase will be *proven* changes how you *cut* it. Write `verify.sh` first and the phase boundaries fall out of it.

**Every clause must be demonstrable from what the agent prints.** The judge has no tools. A clause like ``` `bash verify.sh` exits 0 ``` does not cause anything to run — it's a string the judge hunts for in the transcript. Require the *output*, restated in each turn's final message. This is the single most common way a contract silently degenerates into the agent grading itself.

See `plugin/skills/goalsmith/references/` for the full reasoning:

- `goal-mechanics.md` — the real evaluator contract, verified against the shipped binary
- `verification-ladder.md` — which gate fits which job class
- `templates.md` — the artifact templates
- `challengers.md` — the five adversarial reviewers

## License

MIT
