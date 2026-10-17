# Engineering

Skills I use daily for code work.

- **[codebase-design](./codebase-design/SKILL.md)**: Shared vocabulary for designing a program as an assembly: components with small interfaces and a port per dependency, adapters joining them, contracts, edge components facing foreign components, errors propagated and never wrapped, testable through the constructor and interface. Its topic files build on that vocabulary:
  - **[dependency-injection](./codebase-design/dependency-injection.md)**: How entities get their dependencies: a constructor takes a component's ports and configuration, never creating, fetching or reaching a resource; factories and composites return the interface, only wire, and reach no resource, passing every value a library would read from the environment.
  - **[testing](./codebase-design/testing.md)**: Testing entities through their constructors and interfaces: one behaviour per test, a fake for every port, foreign components through their own test hooks, independent expectations, unit tests by default.
  - **[test-driven-development](./codebase-design/test-driven-development.md)**: Driving development through tests: the red → green → refactor loop, one behaviour per cycle, the constructor, interface and ports evolving with the tests.
  - **[project-structure](./codebase-design/project-structure.md)**: Where code lives on disk: components, adapters, contracts, factories and entrypoints each in their own directory, imports pointing inward, tests mirroring the source tree, legacy code met as a foreign component and moved only along the path of a task.
