---
name: pattern-language
description: A living pattern language for code that an agent writes and a human must still navigate. It keeps code findable by someone who did not write it and cheap to change in directions nobody predicted, while leaving the agent free inside each module, and it improves itself from experience. Use when you write, change, extend or refactor code; when a change does not fit the existing structure; when you are tempted to add a special case, flag or copy to get a task through; when you break a pattern or convention; when you are about to copy the shape of existing code; when you solve the same structural problem a third time; or when you report finished work to the user or check that they still understand the system.
---

# Pattern Language

## Purpose

You write most of the code. The human stays in the loop but cannot read all of it. They still need to find what they are after quickly when it matters, and the code has to keep changing for as long as the project lives.

This skill is a pattern language in Christopher Alexander's sense: a network of patterns, each solving a recurring problem at one scale, applied as the code grows. It is not a rulebook. It shapes how code is found and how it changes. How code is written is left to you.

Three goals drive everything here. Each rule names the goal it serves.

1. **Find.** A human who did not write the code can find the part they need, and the reason it is the way it is, without reading the rest.
2. **Change.** The code stays cheap to change in directions nobody predicted.
3. **Learn.** The language itself improves from what happens when you use it.

## Stance

- **Strict at the skeleton, free in the flesh.** Patterns govern where things live, how boundaries are named and how parts connect. Inside a module or function, write the best code you can, in whatever shape suits it. **Find, Change.**
- **Patterns are defaults, not laws.** Each pattern carries a confidence rating. Break any pattern when it does not serve its goal here, and record why (see [Exceptions](#exceptions)). **Learn.**
- **Apply a pattern only where its problem exists.** A pattern with no problem to solve is clutter: a README nobody needs, an index file for one module, an interface with one implementation. **Find.**
- **Endorsed structure comes first, not merely existing structure.** Existing code is precedent, not permission. In an existing project, follow the conventions the human has endorsed (in the project's `PATTERNS.md`, in its map, or by telling you), then its framework's, then the language's idioms, and only then this skill's seed patterns. Strengthen the structure the project has; do not impose a new one. Before you copy the shape of existing code, check that the shape is endorsed and is not an exception or a graft. **Change:** mistakes do not spread by imitation.
- **Changeability, not generality.** Do not build for futures nobody asked for. An abstraction that bets on the wrong future is as rigid as hard-coding. Keep options cheap instead: clear boundaries, reversible decisions, seams where an extension could go. Generalise on the third real case. **Change.**

## Rates of change

Code is made of layers that change at different rates, like Stewart Brand's shearing layers in buildings:

| Rate | Layer | Examples |
|---|---|---|
| Slowest | Domain concepts | What an invoice *is*; the rules it obeys |
| Slow | Contracts | Public interfaces, APIs, schemas, events, persisted formats |
| Medium | Structure | Modules, boundaries, what depends on what |
| Fast | Implementation | Function bodies, private helpers, internal files |
| Fastest | Configuration | Values, flags, environment |

- Dependencies point from faster layers toward slower ones. Slow code does not depend on fast code. **Change:** fast layers can be rewritten without disturbing what everything else stands on.
- The rate of a change sets how much the human hears about it. You own the fast layers. Tell the human about every change to the slow layers in your [structural report](#structural-report). **Find:** the human reviews the few changes that are expensive to get wrong, not every line.
- Ask before a slow-layer change only when it reverses a recorded decision or something the human said they wanted. Otherwise make it and report it. **Change:** you do not stall on work the human expects you to do.

A project's own layers do not have to match this table. Name them in the project's map.

## The change protocol

Every task changes living code. Follow this sequence, and skip the steps that have nothing to do.

1. **Locate.** Find where the change belongs using the map, the front doors and the names: the same route a human would take. If that was hard, it is a findability failure. Fix the map or the names as part of the task. **Find.**
2. **Fit.** Ask whether the change fits the existing structure. If it does, go to step 4.
3. **Reshape.** If it does not fit, either the structure is wrong for the current intent or you are putting the change in the wrong place. If the structure is wrong, reshape it first, as a separate step that does not change behaviour, so that the change becomes natural. Reshape only as much as the change needs. **Change:** the code always looks designed for what it does now.
4. **Change.** Make the change, freely.
5. **Tidy the map.** If a boundary, contract or domain concept moved, update the map, front doors and decision notes that describe it. Remove what the change made obsolete. **Find.**
6. **Report.** End with a [structural report](#structural-report). **Find.**

### Grafts

A graft gets a task through by working around the structure instead of through it. Each graft is locally reasonable; together they set the code like concrete. Watch for:

- a new flag, mode or special case so that existing code can do a second job
- a copy of existing code with small edits
- reaching into another module's internals instead of going through its front door
- another branch on a type, kind or flag in code that already branches on it
- a catch-all that hides an error so that a test passes
- a new helper that duplicates one that already exists elsewhere

A task is not done when the tests pass. It is done when the change fits. If you graft anyway, because of time, scope or a request for a quick fix, mark it as an exception so it can be found and repaid. **Change, Learn.**

## Exceptions

Break a pattern when following it would make the code worse here. When you do, leave a comment at that place:

```
pattern-exception: <pattern name> — <one-line reason>
```

Keep the marker exact: one search then finds every exception, for a human or a tool. **Find, Learn.**

An exception is not a failure. It is evidence. A pattern that is often broken for good reasons is wrong, or is missing a "not when". **Learn.**

## Evolving the language

The language has shearing layers too: a few high-confidence patterns that change slowly, and candidates at the edge that change fast.

| Rating | Meaning |
|---|---|
| ★★ Core | Fits nearly every codebase. Break it rarely, and always with an exception. |
| ★ Established | Usually right. Break it with an exception when the context differs. |
| ☆ Candidate | A hypothesis. Try it and report how it went. |

The project keeps its own language in `PATTERNS.md` at its root. Create the file the first time the project needs it. It has these sections, each present only when it has content:

- **Local patterns.** Patterns this project has found, in the format of [PATTERNS](PATTERNS.md).
- **Changes to seed patterns.** Seed patterns this project retires, revises or rates differently, each with a reason.
- **Probes.** Findability and changeability questions for evaluation (see [MEASURING](MEASURING.md)).
- **Calibration.** When the human last confirmed each district, and a dated entry for each check-in (see [Calibrating with the human](#calibrating-with-the-human)).
- **Log.** Dated one-line entries for each proposal, revision, promotion and retirement, with its evidence.

How patterns move:

- **Propose** a ☆ candidate when you solve the same structural problem the same way for the third time, or when the human corrects the same kind of thing twice. Add it to `PATTERNS.md` and mention it in your report. **Learn.**
- **Revise** a pattern when its exceptions cluster around one reason. That reason is a missing "not when" or a wrong solution. **Learn.**
- **Promote or retire** a pattern only with the human. Suggest it in a report, with the evidence: how often it was applied, how often it was broken, and why. **Find:** the human keeps control of the slowest layer of the project, its language.

Do not add a pattern to work around a bad one. Fix the bad one or retire it.

## Structural report

End each task that changed more than implementation with a short report on the shape of the system, not its lines. Leave out sections with nothing in them.

```
Structure    – modules or boundaries added, removed, moved, merged or split
Contracts    – public interfaces, schemas, APIs or events changed, and who is affected
Domain       – concepts added, changed or removed
Exceptions   – <pattern> at <location>: <reason>
Language     – patterns proposed or revised, or suggested for promotion or retirement
Findability  – anything that was hard to locate, and what you did about it
Check-in     – when one is due: why, and up to three questions
```

**Find:** the human can follow the system's evolution from the reports alone, and open the code only where a report points.

## Calibrating with the human

You cannot see what the human knows. Every signal in this skill is a guess about their understanding, and the fresh-agent probes are only a proxy for them. Left alone, the guesses drift: reports go unread, the human's picture of the system ages, and you start treating the code as the authority on what is intended. Calibration is how you recover. **Find, Learn:** the human, not the code and not a proxy, stays the measure.

### Check-ins

A check-in is short: a few minutes and at most three questions. There are three kinds:

- **Predict.** Ask the human where something lives, or what happens in some case, before showing them. Compare their answer with the code.
- **Sketch.** Ask them to describe a district or a main path through the system in a sentence or two. Compare it with the map.
- **Confirm intent.** State in two or three lines what you believe they want the system to become, and which patterns you are leaning on most. Ask what is wrong.

Prefer prediction questions: they test the human's picture, not their patience. Ask about what changed since the last check-in, and about districts where slow-layer changes landed.

### When to offer one

Drift sets the cadence, not the calendar. Offer a check-in when:

- several slow-layer changes have landed in a district since the human last confirmed it
- you are about to reshape a district that the human has not confirmed recently
- the human corrected you, or was surprised by the code
- a probe or signal trend changes direction
- you notice you are copying existing code without knowing whether its shape is endorsed

Offer, do not force. Put the offer at the end of a structural report. If the human declines, log it and carry on.

### Reading the gap

Each mismatch is a gap. Decide what kind it is and recover accordingly:

| Gap | What it means | Recovery |
|---|---|---|
| The code is unclear | The human's picture was reasonable; the code does not match it | Reshape or rename the code to match the reasonable picture, or fix the map |
| The human's picture is stale | The code is fine; the human missed changes | Give a short tour of what changed. Make future reports shorter and sharper, because long reports go unread |
| Your picture of intent is wrong | You built in a direction the human did not want | Record the corrected intent in a decision note or `PATTERNS.md`, and revise the patterns you leaned on |
| The proxy is wrong | Fresh-agent probes passed where the human failed, or the reverse | Revise the probes and signals so that they predict the human better |

When you cannot tell, assume the code is unclear. The code exists to fit the human's understanding, not the other way round.

### Recording

Log each check-in in the project's `PATTERNS.md` under **Calibration**: the date, the questions, whether each answer matched, the kind of gap, and what you did. Keep the date when the human last confirmed each district.

A district with many slow-layer changes since its last confirmation is **dark**. In a dark district, treat existing code as unendorsed precedent and its patterns as less certain until the human confirms it again.

## Measuring alignment

Measure, do not block. A check that blocks growth teaches the agent to work around the check, which is just another graft. [MEASURING](MEASURING.md) covers what to measure and how to build checks that report.

## Files

- **[PATTERNS](PATTERNS.md)**: the seed patterns, the format for writing new ones, and how they look in common languages. Read it when you start on a project and whenever a change does not fit.
- **[MEASURING](MEASURING.md)**: signals and evaluations that show whether code is staying findable and changeable, and how to build tools that measure without blocking. Read it when you build or run checks.
