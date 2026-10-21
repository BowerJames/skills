# Measuring

Measurement shows whether code is staying findable and changeable under [pattern-language](SKILL.md). It informs decisions. It does not gate them.

## Principles

- **Report, do not block.** A check that fails the build trains the agent to satisfy the check instead of the goal.
- **Every check honours exceptions.** A `pattern-exception:` marker silences a check at that place, and the check counts the exception instead. The count is evidence for [evolving the language](SKILL.md#evolving-the-language).
- **Watch trends, not thresholds.** One large file is not a problem. Files growing larger every week might be.
- **Measure goals before patterns.** The patterns are guesses about how to reach the goals. When a pattern's metric and its goal's metric disagree, trust the goal.
- **The human is the ground truth.** Every signal and probe here is a proxy for the human's understanding. When a proxy and a [check-in](SKILL.md#calibrating-with-the-human) disagree, the check-in wins and the proxy changes.
- **Few hard invariants, chosen by the human.** If the human wants something to fail the build, it goes in the project's configuration with a reason. Do not add hard checks on your own.

## Signals

These are cheap enough to run on every change.

| Signal | What it suggests | Goal |
|---|---|---|
| Imports that bypass a district's front door | A boundary is leaking | Find, Change |
| A dependency from a slower layer onto a faster one | The foundation is coupling to volatile code | Change |
| Commits that touch many districts (from git history) | A boundary is in the wrong place | Change |
| Distant files that keep changing together (co-change) | Locality of Change is missing | Change |
| Another branch on a type, kind or flag that is already branched on | A graft | Change |
| Duplicated blocks | A graft, or Third Time is due | Change |
| Abstractions with one user | Speculative generality | Change |
| Districts without a purpose statement, or a map naming things that do not exist | The map is stale | Find |
| Exceptions per pattern, grouped by reason | A pattern needs revising | Learn |
| Slow-layer changes per district since the human last confirmed it | The district is going dark | Find |
| New code whose shape matches an exception or an unendorsed precedent | Drift by imitation | Change |

Use existing tools where they fit, for example: import-linter (Python), dependency-cruiser (JavaScript and TypeScript), `internal/` packages and visibility rules (Go, Rust), jscpd (duplication in many languages), and `git log --name-only` for change spread and co-change.

## Evaluations

These run periodically, not on every change. Use a fresh agent with no context as a stand-in for the human who did not write the code. The agent that wrote the code is a poor judge of its own findability.

- **Findability probes.** Questions such as "where is the retry limit for payment webhooks set, and why is it that value?" Measure how many files the fresh agent opens before it answers correctly, and whether it found the *why* as well as the *where*.
- **Changeability probes.** A realistic new requirement. Ask the fresh agent for a plan only, then count the districts, contracts and domain concepts the plan touches.

Keep the probes in the project's `PATTERNS.md` and reuse them, so that results can be compared over time. The best probes are questions the human really asked. Add one each time the human had trouble finding something.

### Calibrating the probes

A fresh agent is not the human. It reads faster, searches better and does not forget. So compare probe results with [check-in](SKILL.md#calibrating-with-the-human) results from time to time:

- **The probe passes and the human fails.** The probe is too easy, or it tests something the human never uses. Rewrite it from the human's wrong answer.
- **The probe fails and the human passes.** The human knows something the code does not show. Ask what, and put it in the code, the map or a decision note.
- **Both agree.** The proxy is still good, so trust its trend until the next comparison.

## Reporting

Report results with the same headings as the [structural report](SKILL.md#structural-report), plus the probe trends. Say what moved since the last run and what you suggest, not every number.
