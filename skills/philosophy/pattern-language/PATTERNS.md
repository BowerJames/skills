# Patterns

The seed language for [pattern-language](SKILL.md). Each pattern solves one recurring problem at one scale, and links to the larger patterns it helps complete and the smaller ones that complete it.

Start at the largest scale a task touches and work down. In a small project most of these have no problem to solve yet (see [Piecemeal Growth](#piecemeal-growth-)).

A project can retire, revise or re-rate any of these in its own `PATTERNS.md`.

## Format

```
### <Name> <rating>

Scale · Rate · Serves

**Problem.**   The forces in tension.
**Pattern.**   What should be true, stated as an outcome, not a procedure.
**Freedom.**   What the pattern deliberately leaves to you.
**Not when.**  Where it does not apply.
**Signal.**    How a human would see that it works.
**Links.**     ↑ larger patterns · ↓ smaller patterns
**In practice.** How it looks in common languages, only where they differ.
```

Write local patterns the same way. Keep each one short. If a pattern needs a page, it is probably two patterns.

## System scale

### Map at the Top ★★

System · Structure · Find

**Problem.** A stranger opening the repo needs to know where to start. Detailed documentation goes stale and nobody reads it.

**Pattern.** The repo root has a short map, either `ARCHITECTURE.md` or a section of the README. It names each district and its purpose, the main paths through the system, and where the hubs are (routing, wiring, configuration). It describes only what changes rarely, and it names things instead of linking to lines, so it stays true.

**Freedom.** Format, length, and whether to include a diagram.

**Not when.** Tiny projects, where the README already is the map.

**Signal.** A stranger can answer "where would X live?" from the map alone.

**Links.** ↓ Districts by Domain, Front Door, Why at the Point of Surprise

### Districts by Domain ★

System · Structure · Find, Change

**Problem.** Layouts organised by technical layer (`controllers/`, `models/`, `utils/`) scatter one feature across the tree. A reader looking for "billing" has to visit everything.

**Pattern.** Top-level divisions name what the system does. Technical layering, if there is any, happens inside a district.

**Freedom.** What counts as a district and how each one is organised inside.

**Not when.** The framework dictates the layout (follow the framework). A library whose domain *is* a technical concern. A project with one domain.

**Signal.** The top-level folder names read like a description of the product.

**Links.** ↑ Map at the Top · ↓ Front Door, Locality of Change

### Shearing Layers ★★

System · Structure · Change

**Problem.** When slow code depends on fast code, every quick change ripples into the foundation, and the foundation sets.

**Pattern.** Dependencies point toward slower layers: implementation depends on contracts, and contracts depend on domain concepts, never the reverse. Volatile choices such as vendors, frameworks and storage sit behind something stable.

**Freedom.** How many layers there are, where the lines fall, and how they are expressed.

**Not when.** Throwaway scripts and prototypes that the human has said will not live long.

**Signal.** Replacing a vendor or framework touches one edge of the system, not its middle.

**Links.** ↓ Explicit Edges, Seams Not Frameworks

## District scale

### Front Door ★★

District · Contract · Find

**Problem.** Readers need a district's purpose and public surface fast. Authors need freedom inside it.

**Pattern.** Each district has one obvious entry that says in a few sentences what the district is for, and exposes what other code may use. Everything else is internal.

**Freedom.** Internal layout, naming and decomposition.

**Not when.** Single-file modules. The framework already defines the entry.

**Signal.** A stranger opening the directory knows which file to read first, and code elsewhere imports only from that file.

**Links.** ↑ Districts by Domain · ↓ Greppable Names, Explicit Edges, Behaviour Has a Witness

**In practice.**
- Python: `__init__.py` with a module docstring that re-exports the public names; internal modules start with `_`.
- TypeScript: `index.ts` re-exporting the public surface, or the `exports` field for a package. Avoid barrels below district level: they hide where things live and slow bundlers.
- Go: the package is the district. A package comment (often in `doc.go`) gives the purpose, exported identifiers are the surface, and `internal/` holds the rest.
- Rust: a `//!` doc comment in `lib.rs` or `mod.rs`; `pub` for the surface and `pub(crate)` for the rest.

### Explicit Edges ★★

District · Structure · Find, Change

**Problem.** Side effects (I/O, network, database, clock, randomness, environment) are where most surprises and failures come from, and hidden ones are hard to find.

**Pattern.** Side effects happen at visible boundaries. Logic that makes decisions can be read and tested without the world it acts on.

**Freedom.** The mechanism: passing functions in, interfaces, ports and adapters, effect systems, or just a plain module of I/O functions.

**Not when.** Scripts whose whole job is the side effect.

**Signal.** Asked "what talks to the database?", a human finds a short list.

**Links.** ↑ Shearing Layers · ↓ Errors Carry Their Origin

### Locality of Change ★

District · Structure · Change

**Problem.** If one change of intent needs edits in many places, every change is costly and some places get missed.

**Pattern.** Things that change together live together. If commits keep touching the same set of distant files, the boundary is in the wrong place, so move it.

**Freedom.** Whether locality comes from colocating, extracting a shared module or merging.

**Not when.** Concerns that genuinely cut across everything, such as logging or auth checks. Give those one home and keep their footprint at each call site light.

**Signal.** A typical commit touches one district.

**Links.** ↑ Districts by Domain · ↓ Third Time, Name It

### Seams, Not Frameworks ★

District · Structure · Change

**Problem.** Code needs to stay open to extension, but building the extension mechanism before the extension exists means guessing its shape, and the guess is usually wrong.

**Pattern.** Leave seams: places where a variation could plug in cheaply later, such as a function parameter, one interface at a boundary or a single dispatch point. Do not build registries, plugin systems or configuration for variations that do not exist yet.

**Freedom.** The form each seam takes.

**Not when.** The human asked for the extension point, or the third real variant has arrived. Then build it.

**Signal.** Adding the second variant is a small, local change, and there is no machinery with only one user.

**Links.** ↑ Shearing Layers · ↓ Third Time, Name It

## Code scale

### Greppable Names ★★

Code · Implementation · Find

**Problem.** Search is the fastest way for a stranger to find code. Names that are generic, built at runtime or reused for different things defeat it.

**Pattern.** Anything a human might search for has a distinct, specific name that appears literally where it is defined and where it is used. Dispatch by string, reflection and metaprogramming stay in hubs, where they are documented.

**Freedom.** Naming style and length, and local variable names.

**Not when.** Small private scopes where the context makes the meaning clear.

**Signal.** Searching for a concept's name finds its definition and its uses, and little else.

**Links.** ↑ Front Door

### Why at the Point of Surprise ★★

Code · Implementation · Find

**Problem.** Code shows what it does and how. The reason for a non-obvious choice lives in someone's head, and for an agent, in a session that has ended.

**Pattern.** Wherever code would surprise a careful reader (a workaround, an odd constant, an order that matters, an obvious approach that was rejected), a short comment says why. A decision that spans districts gets a short decision note, linked from the map or the front door it affects.

**Freedom.** Comment style, and where decision notes live and how they look.

**Not when.** The code is unsurprising. Comments that restate the code are noise.

**Signal.** A stranger asking "why is it 7?" finds the answer next to the 7.

**Links.** ↑ Map at the Top

### Errors Carry Their Origin ★

Code · Implementation · Find

**Problem.** When something fails, the human usually starts from an error message, and a bare "invalid input" leads nowhere.

**Pattern.** An error says what failed, with what input and where: enough to lead a reader to the code. An error crossing a boundary keeps its cause.

**Freedom.** Error types, wrapping style and logging approach.

**Not when.** Messages shown to end users, which must not leak internals. Keep that context in the logs instead.

**Signal.** From the error alone, a human can find the code that raised it.

**Links.** ↑ Explicit Edges

**In practice.** Python `raise ... from err`; Go `fmt.Errorf("...: %w", err)`; Rust `#[source]` with `thiserror`, or `.context()` with `anyhow`; TypeScript `new Error(message, { cause })`.

### Behaviour Has a Witness ★

Code · Contract · Find, Change

**Problem.** The human cannot read all the code, but needs to know what it does and that it still does it.

**Pattern.** Each behaviour the human cares about has a test whose name states the behaviour in plain words. Taken together, the test names read as a spec.

**Freedom.** Test framework, granularity and style.

**Not when.** Exploratory code that the human has not decided to keep.

**Signal.** A human can list a district's test names and learn what it promises.

**Links.** ↑ Front Door

## Growth

### Grow, Don't Graft ★★

All scales · All rates · Change

**Problem.** A change that does not fit the structure can be forced in quickly, and each forced change makes the next one harder.

**Pattern.** Make the change easy, then make the easy change. Reshape the structure without changing behaviour until the change fits, then make it. This is the change protocol in [SKILL](SKILL.md#the-change-protocol).

**Freedom.** How much to reshape: as little as the change needs.

**Not when.** The human asks for a quick fix. Then graft, and mark the exception.

**Signal.** The code reads as if it were designed for what it does now.

**Links.** ↓ every pattern

### Precedent Is Not Permission ★★

All scales · All rates · Change, Find

**Problem.** Agents learn from the code around them. Copying the nearest example spreads its mistakes, exceptions and grafts, until the code becomes its own authority and drifts away from what the human intended.

**Pattern.** New code follows endorsed shapes. Existing code is a template only when its shape is endorsed: recorded in the project's `PATTERNS.md` or map, confirmed by the human, or consistent across its district with no exceptions. Code marked with a `pattern-exception:` is never a template. In a [dark district](SKILL.md#recording), check with the human before you spread a shape further.

**Freedom.** Which endorsed example to follow, and everything inside the implementation.

**Not when.** The human asks you to match specific code.

**Signal.** Exceptions and grafts do not multiply. New code in a district looks like its best code, not its nearest code.

**Links.** ↑ Grow, Don't Graft

### Third Time, Name It ★

All scales · Structure · Change, Learn

**Problem.** Abstracting on the first case is a guess. Never abstracting means copies.

**Pattern.** Tolerate the second similar thing. On the third, extract what they share, whether that is an abstraction, a module or a new candidate pattern in the project's `PATTERNS.md`.

**Freedom.** What to extract and how.

**Not when.** The copies are alike by coincidence and will diverge.

**Signal.** Every abstraction has at least three users, or a recorded reason why not.

**Links.** ↑ Seams Not Frameworks, Locality of Change

### Piecemeal Growth ★

All scales · All rates · Find, Change

**Problem.** Full structure on a small project is clutter. No structure on a large one is a maze.

**Pattern.** Each pattern switches on when its problem appears: a map once there is more than one district, a front door once a district is used by others or grows past a handful of files, a decision note once a decision spans districts. Structure grows with the code, in small steps.

**Freedom.** The thresholds.

**Signal.** Every piece of structure is doing work.

**Links.** ↓ every pattern
