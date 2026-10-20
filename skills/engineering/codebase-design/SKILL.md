---
name: codebase-design
description: The user's prescribed style to design, write, change and test code so that a human can read, verify and change code that an agent wrote. Use when the user wants to design or implement changes in the codebase.
---

# Codebase Design

## Purpose

Agents write code fast. Humans read code slowly. Humans own the intent of the code. Thus the human must understand code that the human did not write.

This skill makes code easy for a human to read and to keep reading as it changes. The human must be able to:

1. Find the code for a given behaviour.
2. Know what the code can do and what it cannot do, without reading the implementation.
3. Estimate the cost of a future change.
4. Verify that a change does what the human intended.

Keep two quantities small: the code that a reader must understand, and the change that a reader must verify.

The user prescribes the style in this skill. Thus the style in this skill is the style that the user knows. Use the structures, names and idioms in this skill and its files. Use each term with the meaning in the glossary. Do not use a different structure or a new term when this skill gives one.

After a change, the code must look as if it was designed for the current intent from the start. Change existing code when the intent requires it. Remove code that the intent no longer requires.

Do refactors and behaviour changes as separate steps. A refactor does not change the tests. A behaviour change is small and local.

Get flexibility from structure. Do not write code for a change that nobody has asked for.

When a choice makes code easier to write but harder to read, choose the easier read.
