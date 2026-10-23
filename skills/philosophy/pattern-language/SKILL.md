---
name: pattern-language
description: How to grow, use and review a project's pattern language together with the human developer. Use when you write, change or refactor code; when a change does not fit the project's patterns; when you break a pattern; when you are about to copy the shape of existing code; when you solve the same structural problem a third time; when the human corrects you or states how they want things done; or when you report finished work to the user.
---

# Pattern Language

## Purpose

Coding agents have written most of the code. The human stays in the loop but cannot read all of it. They still need to find what they are after quickly when it matters, understand what they find and trust the changes you make are aligned with their intentions.

The human is there every step of the way and provides the vision, coding agents maintain context only as long as their session is active. They need to be consistent and be able to retain knowledge of standard and what is important across session.

A pattern language makes that possible. In Christopher Alexander's sense, it is a small network of patterns, each solving one recurring problem at one scale, that the people building something share. When the human and coding agents share the project's language, the human can predict code they have never read and communicate accurately with you about what they want.The ai can see the patterns and expectations that have been set in previous sessions. Both stand to gain from the language.

This skill does not give you a language. Each project grows its own, with the human, from the work. This skill is how you use it, manage it and review it.

Three goals drive everything here. Each rule names the goal it serves.

1. **Find.** A human who did not write the code can find the part they need, and the reason it is the way it is, without reading the rest.
2. **Align.** A human and you can stay aligned in meaning, intention and expectations when referring to all elements of the project.
3. **Learn.** The language improves from what happens when you use it, and it stays the human's language as well as yours.

Patterns can involve anything related to developing in the project whether it is coding practices, domain definitions, project structure or anything else.

## What a pattern is

A pattern names a recurring problem, the forces in tension, and an outcome that resolves them. It is written in this format:

```
---
status: trial | accepted | deprecated
---

# <Name>

## Context
The initial pre-conditions or scenario in which the problem arises. It describes the broader environment so the reader knows whether the pattern applies to their current situation.

## Problem
The core difficulty, tension, or conflict that keeps recurring. This section explains why the current situation is unsatisfactory or unstable.

## Forces
The competing, often contradictory pressures that pull the design in different directions (e.g., cost vs. quality, flexibility vs. simplicity). A good pattern explicitly weighs these forces, showing how the proposed solution balances them.

## Solution
The core instruction, structural arrangement, or behavioral rule that resolves the forces. This is usually written in the imperative mood (e.g., "Do this..." or "Arrange the space such that..."). It often includes diagrams, sketches, or code snippets.

## Resulting Context
What the environment looks like after applying the pattern. This includes the benefits gained, any new problems or sub-problems introduced, and the new baseline state ready for the next layer of patterns.

## Not When
Situations identified from experience for which the pattern is unsuitable.

## Related Patterns
↑ [Larger Name](larger.md)
↓ [Smaller Name](smaller.md)
→ Replaced by [New Name](new-name.md)   (only when deprecated, and only if something replaces it)
```

Keep each pattern short. If a pattern needs a page, it is probably two patterns.

Every pattern carries a status:

| Status | Meaning |
|---|---|
| Trial | Agreed with the human to try out. Apply it where it fits and note how it goes. |
| Accepted | The project lives by it. Break it only with an exception. |
| Deprecated | The project is moving away from it. Code that follows it is not a template; move that code to what replaces it, if anything, when you change it anyway. |

**Align:** the patterns are owned by you and the human. You should not edit accepted ones or promote patterns to accepted without their approval.

## What makes a good pattern

A process can grow a language that is consistent and still bad. Hold each pattern to these tests before you propose it or lean on it.

- **It solves a problem the project has.** A pattern with no problem to solve is clutter. **Learn.**
- **A human can explain why it is there.** If the human can not understand why it exists then it is a poor common language. **Learn.** **Align.**
- **It only prescribes what is required to solve the problem.** A pattern should not limit unnecessarilly. It should prescribe only what is necessary to solve the problem it is there to address. The project requires the freedom to live and adapt.

## The project's language

The language lives in `pattern-language/` at the root of the repo, where a human sees it without looking for it. Create it the first time the project needs a pattern.

```
pattern-language/
  <pattern>.md     one file per pattern, named after it in kebab-case
```

The folder is an Obsidian vault as well as plain markdown. Frontmatter properties make the patterns queryable, and links between patterns give the graph and backlinks. Write links as relative markdown links (`[Front Door](front-door.md)`), not wikilinks, so they also work on GitHub, in reviews and in editors. Keep `.obsidian/` out of git. **Find:** the language reads the same wherever the human opens it.

Git history is the log. Each change to the language is its own small commit, so that `git log pattern-language/<pattern>.md` tells that pattern's story. **Learn.**

## Using the language

- **Apply the patterns during your work.** Try to plan your work with the relevant patterns in mind. Within the bounds the patterns set, write the code however you judge best. Where you believe breaking a pattern gives better code, break it and mark an exception. **Find, Align.**
- **The language is the authority, not the code.** Existing code is a template only where it follows the patterns. Code that merely sits nearby, follows no pattern or deprecated pattern, or carries an exception is not a template.
- **When patterns make a change impossible or convoluted.** This is evidence the patterns may need updating or a new pattern is required. Do not dogmatically force a solution. Mark an exception, then continue and raise the it with the human developer once you have completed your work. **Learn.**

### Exceptions

Patterns are a guide, they should not compromise the quality of the code.

Break a pattern when following it would make the code worse. When you do, leave a comment at that place:

```
pattern-exception: <pattern-file-name> — <one-line reason>
```

Keep the marker exact: one search finds every exception, and the name leads to the pattern. An exception is not a failure. It is evidence for the language. Code with an exception is never a template. **Find, Learn.**

## Managing the language

### Starting

**A new project.** Start with an empty `pattern-language/`. Review the [seeds/](seeds/) patterns for the projects chosen language. These are preconfigured patterns that have been approved in other projects and can be a good starting point. Talk each one through with the human and give them the choice to add each one as a trial pattern to the language.

**An existing project.** The language already exists, unwritten, in the code and in the human's head. Draw it out as you work, starting with the part of the code your task touches rather than surveying everything at once, and ask the human what they wish every agent did consistently. For each shape the code repeats, show the human where it appears and where it is broken, at most three at a time, and decide together:

- **Accept** it if it is what the human intends. The existing code is its trial.
- **Trial** it if the human likes it but is not sure it works here.
- **Deprecate** it if it is an accident or legacy the human wants to move away from, so that it stops being copied.
- **Drop** it if it is not worth recording.

When two shapes solve the same problem, the human picks one, and the other is deprecated. **Learn, Align.**

### Growing

Patterns come from evidence, not from taste. You should discus a pattern with the human when:

- you notice the same problem is solved the same way in at least 3 places **Learn.**
- the human corrects the same kind of thing twice **Align.**
- the human tells you how they want something done **Align.**
- a check-in shows a gap that a pattern would have prevented **Find, Learn.**

A pattern can be discussed to be changed when:

- its exceptions cluster around one reason.
- two patterns pull in opposite directions in the same place.
- it is broken more often than followed.

Suggest accepting or deprecating a pattern with the evidence: how often it was applied, how often it was broken, and why. Do not add a pattern to work around a bad one. Fix the bad one or deprecate it.

When the human rejects a trial pattern, delete its file and give their reason in the commit message. Delete a deprecated pattern once no code follows it. **Learn.**

Always discuss a paatern with the human before writing it down **Align.**

## Reviewing the language

A language only one party knows is not shared. You cannot see what the human knows, and left alone, their picture ages while yours becomes the code's. **Find, Learn:** the human, not the code and not a proxy, stays the measure.

### The language report

End each task that touched the code with a short report after your normal response. Leave out lines with nothing in them.

```
Exceptions   – <pattern> at <location>: <reason>
Proposals     – new patterns, and the evidence for each
Revisions      – patterns to change, and why
Suggested    – patterns to accept or deprecate, with the evidence
Didn't fit   – changes the language did not cover, and what you did
Check-in     – when one is due: why, and up to three questions
```

### Check-ins

A check-in is a few minutes and at most three questions:

- **Predict.** Ask the human where something lives, what it represents or what happens in some case, before showing them. Compare their answer with the code.
- **Sketch.** Ask them to describe a pattern in their own words. Compare it with its file.
- **Confirm intent.** State in two or three lines what you believe they want the system to become, and which patterns you lean on most. Ask what is wrong.

Prefer prediction questions: they test the human's picture, not their patience. Offer a check-in periodically at the end of a report. Use code related to what you touched but not part of the changes.

Each mismatch is a gap. Decide what kind and recover:

| Gap | Recovery |
|---|---|
| The code does not follow the language | Reshape the code, or record an exception |
| The human's picture is stale | Give a short tour of what changed, and make future reports shorter and sharper |
| The language is wrong about intent | Revise the patterns, with the human |

## Keeping it cheap

The language grows out of the work, not alongside it. Proposals, questions and check-ins ride on the report; never stop a task to hold one. Ask at most three questions at a time. If the human is not engaging with the language, ask less, not more. A process that costs more than the code it protects gets skipped.