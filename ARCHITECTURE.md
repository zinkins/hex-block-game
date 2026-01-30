# Hexagonal Puzzle Game Architecture

## Overview
This architecture uses composition over inheritance, leveraging interfaces, components, services, an event bus, and a lightweight Dependency Injection (DI) container. It promotes modularity, testability, and extensibility without the overhead of ECS, suitable for a small-to-medium game project.

## Core Principles
- **Interfaces First**: Define contracts for all major components to ensure loose coupling.
- **Composition Over Inheritance**: Prefer aggregation over subclassing, but allow inheritance where it simplifies shared behavior (e.g., base figure classes).
- **Components**: Small, focused classes that encapsulate specific responsibilities (e.g., rendering, logic).
- **Services**: Stateless business logic handlers that operate on data structures.
- **Event Bus**: Decouple services via publish/subscribe for events like `LineCleared` or `FigurePlaced`.
- **DI Container**: A simple registry/factory for wiring dependencies, enabling easy mocking and swapping.
- **Immutable State**: Game state is rebuilt on each update for predictability.

## Key Components

## Key Components

### 1. Game State (Data Layer)
- **Description**: Immutable data structures representing the game's state.
- **Interfaces**:
  - `IGameState`: Core state (grid, figures, score, etc.).
  - `IHexGrid`: Hexagonal grid with 61 cells (diameter 9, side 5).
  - `IFigure`: Figure data (shape, position, color, rotation).
  - `HexPosition`: Axial coordinates `{ q: number, r: number }`.
- **Components**:
  - `HexGrid`: Manages 61-cell hexagonal grid (diameter 9, side 5); pure data structure with query methods.
  - `Figure`: Holds figure properties; no logic, just data.
- **Grid Configuration**:
  - Side length: 5 hexes
  - Diameter: 9 hexes
  - Total cells: 61
  - Orientation: pointy-top
- **State Updates**: State is rebuilt on each action (e.g., `placeFigure` returns a new `IGameState`). The Game Loop applies updates and triggers re-renders.
- **Benefits**: Pure data, easy to serialize/test, no side effects.

### 2. Services (Logic Layer)
- **Description**: Pure functions or stateless classes handling game rules.
- **Detailed Interfaces**:
  ```typescript
  interface IPlacementValidator {
      // Проверка: можно ли разместить figure на grid в позиции position
      isValidPlacement(grid: IHexGrid, figure: IFigure, position: HexPosition): boolean;
      // Возвращает все валидные позиции для figure на данном grid
      getValidPlacements(grid: IHexGrid, figure: IFigure): HexPosition[];
  }

  interface ILineDetector {
      detectLines(grid: IHexGrid): HexLine[];
  }

  interface HexLine {
      cells: HexPosition[];
      direction: HexDirection;
      length: number; // Все линии минимум 5 ячеек (для поля диаметром 9)
  }

  interface IScoreManager {
      calculatePlacementScore(figure: IFigure): number; // 10 очков за клетку
      calculateLineScore(lines: HexLine[], combo: number): number; // 100 очков за линию, с бонусом за длину
      calculateLengthBonus(length: number): number; // Бонус за длину: 5 ячеек=база, 7 ячеек=+50%, 9 ячеек=+100%
      resetCombo(): void;
      getComboMultiplier(): number;
  }

  // Scoring Rules (from PRD):
  // - Placement: 10 points per cell
  // - Line destruction: 100 points per line (minimum 5 cells per line)
  // - Length bonus: Longer lines give more points (7 cells: +50%, 9 cells: +100%)
  // - Combo multiplier: x1.5 - x5 for clearing 2+ lines simultaneously
  ```
- **Services**:
  - `FigureGeneratorService`: Implements weighted random generation.
  - `PlacementValidatorService`: Validates against grid boundaries/overlaps.
  - `LineDetectorService`: Scans for lines in 6 directions.
  - `ScoreManagerService`: Handles scoring logic.
  - `StateManagerService`: Central hub for state updates; uses other services and publishes events.
- **Benefits**: Stateless, unit-testable, composable.

### 3. Components (Presentation/Input Layer)
- **Description**: Handle rendering and user input.
- **Interfaces**:
  - `IRenderer`: Abstract rendering (Canvas, WebGL).
  - `IInputHandler`: Processes mouse/touch events.
  - `IDragManager`: Manages drag-and-drop interactions.
- **Components**:
  - `CanvasRenderer`: Renders grid/figures on canvas.
  - `InputHandler`: Translates events to actions.
  - `DragManager`: Tracks drag state and previews.
- **Benefits**: Swappable (e.g., switch to SVG renderer).

### 4. Game Loop (Orchestration Layer)
- **Description**: Central coordinator using DI to wire everything.
- **Interfaces**:
  - `IGameLoop`: Manages update/render cycle at fixed FPS (e.g., 60).
  - `IStateManager`: Applies actions to state (e.g., `placeFigure`, `rotateFigure`).
- **Components**:
  - `GameLoop`: Runs the loop, calls `update()` and `render()`.
  - `StateManager`: Delegates to services (e.g., `PlacementValidator`) and rebuilds state.
- **Flow**:
  1. Input → `InputHandler` → emits action (e.g., `FigurePlacedAction`).
  2. `StateManager` validates via services → returns new state.
  3. `GameLoop` triggers `Renderer` to draw updated state.
  4. Services publish events (e.g., `LineCleared`) → subscribers (e.g., `AnimationService`) act.
- **DI Container**: Simple class with register/resolve methods for injecting dependencies.

### 5. Animations
- **Description**: Handle visual effects (e.g., line destruction, figure placement).
- **Interface**:
  - `IAnimationService`: Queues and updates animations based on game state.
- **Implementation**:
  - Animations are separate from game logic (state-based).
  - Subscribe to events (e.g., `LineCleared`) to start animations.
  - Animation frames interpolate values; no logic in render loop.
- **Benefits**: Decouples visual effects from core game rules.

## Dependency Injection
- Use a lightweight DI container (e.g., custom `DIContainer` class).
- Register interfaces to implementations at startup.
- Inject services into components (e.g., `GameLoop` gets `IScoreManager` via DI).
- Enables testing: Mock services for unit tests.
- **Scope**: Prefer constructor injection for testability. Use a singleton only for the DI container itself.

## Event Bus
- **Purpose**: Decouple services via publish/subscribe. Services emit events (e.g., `LineCleared`) without knowing subscribers.
- **Interface**:
  - `IEventBus`: `publish(event: string, data: any)`, `subscribe(event: string, callback: (data) => void)`.
- **Event Types**:
  - `LineCleared`: Emitted when lines are detected for destruction. Data: `{ lines: HexLine[], combo: number }`.
  - `FigurePlaced`: Emitted when a figure is successfully placed. Data: `{ figure: IFigure, position: HexPosition }`.
  - `FigurePicked`: Emitted when a user picks up a figure to drag. Data: `{ figure: IFigure, slotIndex: number }`. Useful for sound/visual feedback.
  - `GameOver`: Emitted when no valid moves remain. Data: `{ finalScore: number, highScore: boolean }`.
- **Example Flow**:
  1. `LineDetectorService` finds a line → publishes `LineCleared` with line data.
  2. `ScoreManagerService` subscribes to `LineCleared` → updates score.
  3. `AnimationService` subscribes to `LineCleared` → triggers destruction animation.
  4. `DragManager` publishes `FigurePicked` → `SoundManager` plays "pick" sound.
- **Benefits**: Loosely coupled; add new behavior without modifying existing services.

## Example Wiring (Pseudocode)
```typescript
// Setup DI container
const di = new DIContainer();

// Register core
di.register(IEventBus, EventBus);
di.register(IGameLoop, GameLoop);
di.register(IStateManager, StateManager);

// Register services
di.register(IFigureGenerator, FigureGeneratorService);
di.register(IPlacementValidator, PlacementValidatorService);
di.register(ILineDetector, LineDetectorService);
di.register(IScoreManager, ScoreManagerService);
di.register(IAnimationService, AnimationService);

// Register components
di.register(IRenderer, CanvasRenderer);
di.register(IInputHandler, InputHandler);

// Wire event subscriptions (at startup)
const eventBus = di.resolve(IEventBus);
const scoreManager = di.resolve(IScoreManager);
const animationService = di.resolve(IAnimationService);
eventBus.subscribe('LineCleared', (data) => scoreManager.onLineCleared(data));
eventBus.subscribe('LineCleared', (data) => animationService.playDestruction(data));

// Start game
const gameLoop = di.resolve(IGameLoop);
gameLoop.start();
```

## Extensibility
- Add new features by implementing new services (e.g., `IPowerUpService` for bonuses).
- Add new events for communication (e.g., `PowerUpActivated`).
- Swap components (e.g., `WebGLRenderer` instead of `CanvasRenderer`).
- No refactoring needed for new figure types—just extend data/services.
- Add new game modes by implementing new `IGameMode` service or extending `IStateManager`.

## Testability
- **Services**: Pure functions—test with input/output assertions.
- **State**: Immutable—snapshot tests for state transitions.
- **Components**: Mock via DI (e.g., mock `IRenderer` for `GameLoop` tests).
- **Events**: Test event publishing/subscribing with mock event bus.
- **Integration**: Test full workflows (e.g., "place figure" → "score update") via `StateManager`.

## File Structure
```
src/
├── interfaces/          # All interface definitions
├── services/            # Service implementations (logic)
├── components/          # Component classes (rendering, input)
├── models/              # Data structures (game state, figures)
├── events/              # Event types and bus implementation
├── animations/          # Animation service and effects
├── di/                  # DI container
└── main.ts              # Entry point with wiring
```

## Pitfalls to Avoid
1. **Over-Engineering**: The DI container and event bus add abstraction. Keep them simple; avoid " DI for everything."
2. **State Bloat**: Don't let `IGameState` grow too large. Split into sub-states (e.g., `IGridState`, `IScoreState`) if needed.
3. **Circular Dependencies**: Services may need references (e.g., `ScoreManager` → `EventBus`). Avoid bidirectional dependencies.
4. **Animation-Logic Coupling**: Keep animations visual-only; game logic should not depend on animation completion.
5. **Over-Abstracting Rendering**: Start with `CanvasRenderer`; add `WebGLRenderer` only if performance demands it.

## Tradeoffs
- More boilerplate than inheritance OO, but less risk of tight coupling.
- DI and event bus add learning curve but improve testability and extensibility.
- Benefits: High modularity, easy refactoring, scalable for future features (e.g., power-ups, multiplayer).