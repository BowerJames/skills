# Engineering

Skills I use daily for code work.

- **[codebase-design](./codebase-design/SKILL.md)**: Shared vocabulary for designing modules: small interfaces, ports declared in the constructor, testable through the constructor and interface.
- **[dependency-injection](./dependency-injection/SKILL.md)**: What a constructor receives: a module depends on its own ports, and ports and configuration go in the constructor, not the interface.
- **[factory-pattern](./factory-pattern/SKILL.md)**: The code that calls a module's constructor and returns the entity: return the interface, hold no behaviour, reach resources here and pass them on.
- **[testing](./testing/SKILL.md)**: Testing entities through their constructors and interfaces: one behaviour per test, a test adapter for every port, independent expectations.
- **[test-driven-development](./test-driven-development/SKILL.md)**: Driving development through tests: the red → green loop, one behaviour per cycle, the constructor and interface evolving with the tests.
- **[project-structure](./project-structure/SKILL.md)**: Where code lives on disk: modules and data carriers in the core, adapters, factories and entrypoints in the infrastructure, imports pointing inward, tests mirroring the source tree.
