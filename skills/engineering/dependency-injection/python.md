# Dependency Injection in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling. The vocabulary's Python shapes — Protocols and Callables as interfaces, the three shapes of a seam — live in [codebase-design's python.md](../codebase-design/python.md).

## Testing: fixtures wire the graph

A pytest fixture that assembles the graph with fakes is the test doing its own wiring — the same discipline at test scope. A fake is a small class satisfying the Protocol; a seam needs no mock library. `monkeypatch` is a service locator for tests: injecting a fake at the seam beats patching a global every time.

## Gotchas

- **Mutable default arguments** — `def __init__(self, hooks=[])` shares one list across every instance. Defaults must be immutable; use `None` and construct inside.
- **No framework needed.** `dependency-injector` and friends automate wiring; the discipline is stdlib-only — typed parameters and Protocols carry it.
