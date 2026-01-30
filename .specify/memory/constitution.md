<!-- 
Sync Impact Report:
- Version change: 0.1.0 → 1.0.0 (MAJOR: New constitution established)
- Added sections: Core Principles (5), Performance Standards, Development Workflow, Governance
- Removed sections: None
- Templates requiring updates: ✅ plan-template.md, ✅ spec-template.md, ✅ tasks-template.md
- TODO: None
-->

# Hex Puzzle Game Constitution

## Core Principles

### I. Interface-First Architecture
Define interfaces for all major components first to ensure loose coupling. Prefer composition over inheritance, use immutable state for predictability, and implement a lightweight DI container for dependency management. Services must be stateless and independently testable. Components must be swappable (e.g., Canvas vs WebGL renderer).

### II. Test-Driven Development (NON-NEGOTIABLE)
Mandatory TDD: Tests written → User approved → Tests fail → Then implement. Red-Green-Refactor cycle strictly enforced. All services must be unit tested with input/output assertions. State transitions must be snapshot tested. Integration tests required for new contracts, state changes, and event-driven workflows.

### III. Rapid Iteration Ready
Embrace frequent design changes through modular services, event-driven architecture, and easy extensibility. New features must be implementable without refactoring existing code. Services publish events (LineCleared, FigurePlaced) without knowing subscribers. DI container enables easy mocking and swapping for rapid prototyping.

### IV. Consistent User Experience
Implement unified design system with responsive input handling and predictable game mechanics. Accessibility standards mandatory. Input handling must work across mobile and desktop. Visual feedback must be immediate and consistent. Color scheme and animations must follow established patterns.

### V. Performance-First Rendering
Maintain 60 FPS target with efficient canvas/WebGL rendering. Minimize state rebuilds through strategic immutability. Optimize hex grid calculations and collision detection. Animation service must be separate from game logic. Memory usage must be predictable and bounded for long sessions.

## Performance Standards

### Rendering Requirements
- Target: 60 FPS consistently
- Canvas rendering optimized for hexagonal grid calculations
- WebGL renderer optional for complex visual effects
- Animation system must not block game logic threads
- Memory usage: <100MB for typical gameplay sessions

### Game Performance
- Hex grid validation: <16ms per operation
- Line detection: <33ms per frame
- Figure placement validation: <8ms per operation
- State rebuilds: Optimized for minimal allocations
- Event bus: Subscriptions must not impact main loop performance

### Load Performance
- Initial load time: <3 seconds on modern hardware
- Asset loading: Progressive with visible feedback
- No blocking resources during gameplay
- Graceful degradation for lower-end devices

## Development Workflow

### Code Quality Gates
- All new code must have corresponding tests (TDD)
- Code coverage minimum: 80% for services, 70% for components
- Linting and type checking must pass before PR submission
- Complexity metrics: Cyclomatic complexity <10 per function, cognitive complexity <15

### Review Process
- PR must include test plan and performance impact assessment
- Architecture changes require diagram and rationale documentation
- User experience changes must be tested on target devices
- Performance regressions must be justified and documented

### Quality Assurance
- Automated testing: Unit tests + Integration tests + E2E tests
- Manual testing required for: UI interactions, touch/mouse handling, animations
- Performance benchmarking required for: New features, optimizations, platform changes
- Accessibility testing: Keyboard navigation

## Governance

### Amendment Procedure
- Amendments require documentation, rationale, and migration plan
- Major changes (version bump) require team approval
- Changes must not break existing contracts without version bump
- Update templates and documentation simultaneously with amendments

### Versioning Policy
- MAJOR: Backward incompatible governance/principle removals or redefinitions
- MINOR: New principle/section added or materially expanded guidance  
- PATCH: Clarifications, wording, typo fixes, non-semantic refinements
- Semantic versioning: MAJOR.MINOR.PATCH format for all artifacts

### Compliance Review
- All PRs/reviews must verify constitution compliance
- Complexity must be justified with documentation
- Performance impact must be measured and documented
- Use ARCHITECTURE.md for runtime development guidance
- Use PRD.md for product requirements and game design guidance

**Version**: 1.0.0 | **Ratified**: 2025-01-30 | **Last Amended**: 2025-01-30
