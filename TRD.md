# Technical Requirements Document (TRD)
# Hexagonal Puzzle Game

**Версия:** 1.0
**Дата:** 20.01.2026
**Статус:** Draft
**Зависимый документ:** PRD v1.0

---

## 1. Обзор документа

### 1.1 Цель документа

Данный документ описывает техническую архитектуру, структуру кода, алгоритмы и технические решения для игры Hexagonal Puzzle Game. Документ служит руководством для разработчиков и определяет стандарты кодирования, организацию модулей и требования к производительности.

### 1.2 Технологический стек

**Язык программирования:** TypeScript 5.x (с строгой типизацией)
**Сборка:** Vite 7.x
**Рендеринг:** HTML5 Canvas API (2D контекст)
**Упаковка:** Rollup (ES modules, tree shaking)
**Целевые платформы:** Современные браузеры (ES2020+)

### 1.3 Целевые показатели производительности

| Метрика | Целевое значение |
|---------|------------------|
| Время загрузки (3G) | < 1.5 секунды |
| Размер билда (gzipped) | 20-30 KB |
| FPS | стабильные 60 FPS |
| Память (RAM) | < 50 MB |
| Время отклика UI | < 100 ms |

---

## 2. Архитектура приложения

### 2.1 Модульная структура

```
src/
├── core/
│   ├── Game.ts              # Главный игровой контроллер
│   ├── GameLoop.ts          # Игровой цикл (60 FPS)
│   ├── StateManager.ts      # Управление состояниями игры
│   └── EventBus.ts          # Событийная шина
├── grid/
│   ├── HexGrid.ts           # Гексогональная сетка
│   ├── HexCell.ts           # Отдельная ячейка
│   ├── HexCoord.ts          # Система координат
│   └── LineDetector.ts      # Определение линий
├── figures/
│   ├── Figure.ts            # Базовый класс фигуры
│   ├── FigureGenerator.ts   # Генерация фигур
│   ├── FigureRotator.ts     # Вращение фигур
│   └── FigureShapes.ts      # Константы форм фигур
├── game/
│   ├── ScoreManager.ts      # Подсчёт очков
│   ├── ComboSystem.ts       # Система комбо
│   ├── PlacementValidator.ts # Валидация размещения
│   └── LineDestroyer.ts     # Уничтожение линий
├── input/
│   ├── InputHandler.ts      # Обработка ввода
│   ├── TouchHandler.ts      # Touch события
│   ├── MouseHandler.ts      # Mouse события
│   └── DragManager.ts       # Drag-and-drop логика
├── renderer/
│   ├── CanvasRenderer.ts    # Основной рендерер
│   ├── HexRenderer.ts       # Рендеринг гексов
│   ├── FigureRenderer.ts    # Рендеринг фигур
│   └── UIRenderer.ts        # Рендеринг UI
├── effects/
│   ├── ParticleSystem.ts    # Система частиц
│   ├── AnimationManager.ts  # Менеджер анимаций
│   └── SoundManager.ts      # Звуковой менеджер
├── ui/
│   ├── UIManager.ts         # Менеджер UI
│   ├── screens/             # Экраны
│   │   ├── MainMenu.ts
│   │   ├── GameScreen.ts
│   │   ├── GameOverScreen.ts
│   │   └── SettingsScreen.ts
│   └── components/          # UI компоненты
│       ├── Button.ts
│       ├── ScoreDisplay.ts
│       └── FigureSlots.ts
├── analytics/
│   ├── Analytics.ts         # Интерфейс аналитики
│   ├── GameAnalytics.ts     # GameAnalytics интеграция
│   └── EventTracker.ts      # Отслеживание событий
├── platform/
│   ├── PlatformDetector.ts  # Определение платформы
│   ├── PlayGamaAdapter.ts   # PlayGama интеграция
│   └── SaveSystem.ts        # Система сохранений
├── utils/
│   ├── MathUtils.ts         # Математические утилиты
│   ├── ColorUtils.ts        # Работа с цветами
│   └── Constants.ts         # Константы игры
├── main.ts                  # Точка входа
└── styles.css               # Стили (минимум)
```

### 2.2 Архитектурный паттерн

**Entity-Component-System (ECS) упрощённый:**
- Game — главный контроллер
- Модули взаимодействуют через EventBus
- Строгое разделение логики и рендеринга

**Однонаправленный поток данных:**
```
Input → EventBus → Logic → State → Renderer → Canvas
                    ↓
              Analytics
```

---

## 3. Система координат и гексогональная сетка

### 3.1 Типы координат

```typescript
// Осевые координаты (Axial)
interface AxialCoord {
    q: number;  // column
    r: number;  // row
}

// Кубические координаты (Cube)
interface CubeCoord {
    q: number;
    r: number;
    s: number;  // s = -q - r
}

// Пиксельные координаты
interface PixelCoord {
    x: number;
    y: number;
}
```

### 3.2 Конвертация координат

```typescript
class HexCoord {
    static axialToCube(axial: AxialCoord): CubeCoord {
        return {
            q: axial.q,
            r: axial.r,
            s: -axial.q - axial.r
        };
    }

    static cubeToAxial(cube: CubeCoord): AxialCoord {
        return { q: cube.q, r: cube.r };
    }

    static axialToPixel(axial: AxialCoord, size: number): PixelCoord {
        // Pointy-top orientation
        const x = size * (Math.sqrt(3) * axial.q + Math.sqrt(3) / 2 * axial.r);
        const y = size * (3 / 2 * axial.r);
        return { x, y };
    }

    static pixelToAxial(pixel: PixelCoord, size: number): AxialCoord {
        const q = (Math.sqrt(3) / 3 * pixel.x - 1 / 3 * pixel.y) / size;
        const r = (2 / 3 * pixel.y) / size;
        return this.roundAxial({ q, r });
    }

    static cubeDistance(a: CubeCoord, b: CubeCoord): number {
        return (Math.abs(a.q - b.q) +
                Math.abs(a.q + a.r - b.q - b.r) +
                Math.abs(a.r - b.r)) / 2;
    }

    static roundAxial(coord: AxialCoord): AxialCoord {
        const cube = this.axialToCube(coord);
        const roundedCube = this.roundCube(cube);
        return this.cubeToAxial(roundedCube);
    }
}
```

### 3.3 Структура игрового поля

```typescript
class HexGrid {
    radius: number;           // Радиус (3 для 5-клеточного диаметра)
    cells: Map<string, HexCell>;  // Карта ячеек
    center: AxialCoord;

    constructor(radius: number = 3) {
        this.radius = radius;
        this.center = { q: 0, r: 0 };
        this.initializeGrid();
    }

    private initializeGrid(): void {
        for (let q = -this.radius + 1; q < this.radius; q++) {
            const r1 = Math.max(-this.radius + 1, -q - this.radius + 1);
            const r2 = Math.min(this.radius - 1, -q + this.radius - 1);
            for (let r = r1; r <= r2; r++) {
                const key = `${q},${r}`;
                this.cells.set(key, new HexCell({ q, r }));
            }
        }
    }

    getCell(coord: AxialCoord): HexCell | undefined {
        return this.cells.get(`${coord.q},${coord.r}`);
    }

    isValidCoord(coord: AxialCoord): boolean {
        return this.cells.has(`${coord.q},${coord.r}`);
    }

    getNeighbors(coord: AxialCoord): HexCell[] {
        const directions = [
            { q: 1, r: 0 }, { q: 1, r: -1 }, { q: 0, r: -1 },
            { q: -1, r: 0 }, { q: -1, r: 1 }, { q: 0, r: 1 }
        ];
        return directions
            .map(d => this.getCell({ q: coord.q + d.q, r: coord.r + d.r }))
            .filter((c): c is HexCell => c !== undefined);
    }
}
```

### 3.4 Определение линий

```typescript
class LineDetector {
    // 6 направлений для гексогональной сетки
    private static readonly DIRECTIONS: AxialCoord[] = [
        { q: 1, r: -1 },   // Направление 1
        { q: 1, r: 0 },    // Направление 2
        { q: 0, r: 1 },    // Направление 3
        { q: -1, r: 1 },   // Направление 4
        { q: -1, r: 0 },   // Направление 5
        { q: 0, r: -1 }    // Направление 6
    ];

    static findCompleteLines(grid: HexGrid): AxialCoord[][] {
        const completeLines: AxialCoord[][] = [];

        for (const direction of this.DIRECTIONS) {
            const line = this.findLine(grid, direction);
            if (this.isLineComplete(grid, line)) {
                completeLines.push(line);
            }
        }

        return completeLines;
    }

    private static findLine(grid: HexGrid, direction: AxialCoord): AxialCoord[] {
        const line: AxialCoord[] = [];
        // Находим крайние точки в направлении и собираем линию
        // ... (реализация поиска линии от края до края)
        return line;
    }

    private static isLineComplete(grid: HexGrid, line: AxialCoord[]): boolean {
        return line.every(coord => {
            const cell = grid.getCell(coord);
            return cell && cell.isOccupied;
        });
    }
}
```

---

## 4. Игровые фигуры

### 4.1 Структура фигуры

```typescript
enum FigureType {
    MONOMINO = 'monomino',
    TETRAMINO = 'tetromino'
}

class Figure {
    type: FigureType;
    cells: AxialCoord[];       // Относительные координаты
    rotation: number;          // 0-5 (кратные 60°)
    color: string;

    constructor(type: FigureType, cells: AxialCoord[], color: string) {
        this.type = type;
        this.cells = cells;
        this.rotation = 0;
        this.color = color;
    }

    rotate(direction: 'clockwise' | 'counterclockwise'): void {
        const angle = direction === 'clockwise' ? 1 : -1;
        this.rotation = (this.rotation + angle + 6) % 6;

        // Поворот координат на 60°
        this.cells = this.cells.map(cell => ({
            q: -cell.r,
            r: cell.q + cell.r
        }));
    }

    getAbsoluteCoords(origin: AxialCoord): AxialCoord[] {
        return this.cells.map(cell => ({
            q: origin.q + cell.q,
            r: origin.r + cell.r
        }));
    }
}
```

### 4.2 Формы тетрамино

```typescript
interface FigureShape {
    name: string;
    cells: AxialCoord[];
    color: string;
}

const TETRAMINO_SHAPES: FigureShape[] = [
    // Прямая линия (Line)
    {
        name: 'line',
        cells: [{ q: 0, r: 0 }, { q: 1, r: 0 }, { q: 2, r: 0 }, { q: 3, r: 0 }],
        color: '#4ECDC4'
    },
    // Угловая (Corner)
    {
        name: 'corner',
        cells: [{ q: 0, r: 0 }, { q: 1, r: 0 }, { q: 2, r: 0 }, { q: 0, r: 1 }],
        color: '#FF6B6B'
    },
    // Треугольная (Triangle)
    {
        name: 'triangle',
        cells: [{ q: 0, r: 0 }, { q: 1, r: 0 }, { q: 2, r: 0 }, { q: 1, r: 1 }],
        color: '#FFD93D'
    },
    // S-форма
    {
        name: 's',
        cells: [{ q: 1, r: 0 }, { q: 2, r: 0 }, { q: 0, r: 1 }, { q: 1, r: 1 }],
        color: '#6BCB77'
    },
    // Z-форма
    {
        name: 'z',
        cells: [{ q: 0, r: 0 }, { q: 1, r: 0 }, { q: 1, r: 1 }, { q: 2, r: 1 }],
        color: '#9B59B6'
    },
    // Г-образная (Hook)
    {
        name: 'hook',
        cells: [{ q: -1, r: 0 }, { q: 0, r: 0 }, { q: 1, r: 0 }, { q: 1, r: 1 }],
        color: '#3498DB'
    },
    // Удлинённая Г-форма (LongHook)
    {
        name: 'longHook',
        cells: [{ q: -1, r: 0 }, { q: 0, r: 0 }, { q: 1, r: 0 }, { q: 2, r: -1 }],
        color: '#1ABC9C'
    },
    // Трапециевидная (Boat)
    {
        name: 'boat',
        cells: [{ q: -1, r: 0 }, { q: 0, r: -1 }, { q: 1, r: -1 }, { q: 1, r: 0 }],
        color: '#E74C3C'
    },
    // Три луча из центра (Rays)
    {
        name: 'rays',
        cells: [{ q: 0, r: 0 }, { q: 1, r: 0 }, { q: 0, r: -1 }, { q: -1, r: 1 }],
        color: '#FF9F43'
    }
];

const MONOMINO_SHAPE: FigureShape = {
    name: 'monomino',
    cells: [{ q: 0, r: 0 }],
    color: '#FFFFFF'
};
```

**Всего уникальных фигур:** 10 (9 тетрамино + 1 мономино)

### 4.3 Генератор фигур

```typescript
class FigureGenerator {
    private readonly tetrominoWeights: number[] = [1, 1, 1, 1, 1, 1, 1, 1, 1]; // 9 фигур
    private readonly monominoWeight: number = 0.3;

    generateRandom(): Figure {
        const isMonomino = Math.random() < this.monominoWeight;

        if (isMonomino) {
            return this.createMonomino();
        }

        return this.createRandomTetromino();
    }

    private createMonomino(): Figure {
        return new Figure(FigureType.MONOMINO, MONOMINO_SHAPE.cells, MONOMINO_SHAPE.color);
    }

    private createRandomTetromino(): Figure {
        const shapeIndex = this.weightedRandom(this.tetrominoWeights);
        const shape = TETRAMINO_SHAPES[shapeIndex];
        const figure = new Figure(FigureType.TETRAMINO, shape.cells, shape.color);

        // Случайная начальная ориентация
        const rotations = Math.floor(Math.random() * 6);
        for (let i = 0; i < rotations; i++) {
            figure.rotate('clockwise');
        }

        return figure;
    }

    private weightedRandom(weights: number[]): number {
        const total = weights.reduce((a, b) => a + b, 0);
        let random = Math.random() * total;
        for (let i = 0; i < weights.length; i++) {
            random -= weights[i];
            if (random <= 0) return i;
        }
        return 0;
    }
}
```

---

## 5. Система ввода

### 5.1 Обработка ввода

```typescript
interface InputState {
    isDragging: boolean;
    selectedFigure: Figure | null;
    dragStart: PixelCoord | null;
    currentPosition: PixelCoord | null;
    hoverCell: AxialCoord | null;
}

class InputHandler {
    private canvas: HTMLCanvasElement;
    private state: InputState;
    private eventBus: EventBus;
    private touchHandler: TouchHandler;
    private mouseHandler: MouseHandler;

    constructor(canvas: HTMLCanvasElement, eventBus: EventBus) {
        this.canvas = canvas;
        this.eventBus = eventBus;
        this.state = this.createInitialState();
        this.touchHandler = new TouchHandler(this);
        this.mouseHandler = new MouseHandler(this);
    }

    private createInitialState(): InputState {
        return {
            isDragging: false,
            selectedFigure: null,
            dragStart: null,
            currentPosition: null,
            hoverCell: null
        };
    }

    onPointerDown(x: number, y: number): void {
        const cell = this.pixelToCell(x, y);
        this.eventBus.emit(InputEvents.POINTER_DOWN, { x, y, cell });
    }

    onPointerMove(x: number, y: number): void {
        const cell = this.pixelToCell(x, y);
        this.eventBus.emit(InputEvents.POINTER_MOVE, { x, y, cell });
    }

    onPointerUp(x: number, y: number): void {
        const cell = this.pixelToCell(x, y);
        this.eventBus.emit(InputEvents.POINTER_UP, { x, y, cell });
    }

    private pixelToCell(x: number, y: number): AxialCoord | null {
        const size = CONFIG.cellSize;
        return HexCoord.pixelToAxial({ x, y }, size);
    }
}
```

### 5.2 Drag-and-drop логика

```typescript
class DragManager {
    private eventBus: EventBus;
    private previewFigure: Figure | null = null;
    private validPlacement: boolean = false;

    constructor(eventBus: EventBus) {
        this.eventBus = eventBus;
    }

    startDrag(figure: Figure, position: PixelCoord): void {
        this.previewFigure = figure;
        this.eventBus.emit(DragEvents.START, { figure, position });
    }

    updateDrag(position: PixelCoord): void {
        if (!this.previewFigure) return;

        const cell = HexCoord.pixelToAxial(position, CONFIG.cellSize);
        const validPlacement = this.validatePlacement(this.previewFigure, cell);

        this.validPlacement = validPlacement;
        this.eventBus.emit(DragEvents.UPDATE, {
            position,
            cell,
            validPlacement,
            previewFigure: this.previewFigure
        });
    }

    endDrag(position: PixelCoord): void {
        if (!this.previewFigure) return;

        const cell = HexCoord.pixelToAxial(position, CONFIG.cellSize);
        const canPlace = this.validatePlacement(this.previewFigure, cell);

        if (canPlace) {
            this.eventBus.emit(DragEvents.END_SUCCESS, {
                figure: this.previewFigure,
                cell
            });
        } else {
            this.eventBus.emit(DragEvents.END_FAIL, {
                figure: this.previewFigure
            });
        }

        this.previewFigure = null;
    }

    private validatePlacement(figure: Figure, origin: AxialCoord): boolean {
        const coords = figure.getAbsoluteCoords(origin);
        return coords.every(coord => {
            const cell = gameState.grid.getCell(coord);
            return cell && !cell.isOccupied;
        });
    }
}
```

---

## 6. Система рендеринга

### 6.1 Основной рендерер

```typescript
class CanvasRenderer {
    private canvas: HTMLCanvasElement;
    private ctx: CanvasRenderingContext2D;
    private width: number;
    private height: number;
    private scale: number;

    constructor(canvas: HTMLCanvasElement) {
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d')!;
        this.resize();
    }

    resize(): void {
        const dpr = window.devicePixelRatio || 1;
        const rect = this.canvas.getBoundingClientRect();

        this.width = rect.width;
        this.height = rect.height;
        this.scale = dpr;

        this.canvas.width = rect.width * dpr;
        this.canvas.height = rect.height * dpr;

        this.ctx.scale(dpr, dpr);
    }

    clear(): void {
        this.ctx.fillStyle = CONFIG.colors.background;
        this.ctx.fillRect(0, 0, this.width, this.height);
    }

    render(gameState: GameState): void {
        this.clear();
        this.renderGrid(gameState.grid);
        this.renderFigures(gameState.figures);
        this.renderUI(gameState.ui);
        this.renderEffects(gameState.effects);
    }
}
```

### 6.2 Рендеринг гексагональной ячейки

```typescript
class HexRenderer {
    private ctx: CanvasRenderingContext2D;
    private cellSize: number;

    constructor(ctx: CanvasRenderingContext2D, cellSize: number) {
        this.ctx = ctx;
        this.cellSize = cellSize;
    }

    render(cell: HexCell, position: PixelCoord, isValid: boolean = true): void {
        const hexPath = this.createHexPath(position, this.cellSize * CONFIG.hexScale);

        // Заливка
        this.ctx.beginPath();
        this.ctx.fillStyle = isValid ? cell.color : CONFIG.colors.hover;
        this.ctx.fill(hexPath);

        // Граница
        this.ctx.strokeStyle = CONFIG.colors.border;
        this.ctx.lineWidth = 1;
        this.ctx.stroke();

        // Эффекты при наведении
        if (cell.isHovered && isValid) {
            this.ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
            this.ctx.fill(hexPath);
        }

        // Эффекты для занятых ячеек
        if (cell.isOccupied) {
            this.ctx.shadowColor = cell.figureColor;
            this.ctx.shadowBlur = 10;
            this.ctx.fillStyle = cell.figureColor;
            this.ctx.fill(hexPath);
            this.ctx.shadowBlur = 0;
        }
    }

    private createHexPath(center: PixelCoord, size: number): Path2D {
        const path = new Path2D();
        for (let i = 0; i < 6; i++) {
            const angle = (Math.PI / 3) * i - Math.PI / 6;
            const x = center.x + size * Math.cos(angle);
            const y = center.y + size * Math.sin(angle);
            if (i === 0) path.moveTo(x, y);
            else path.lineTo(x, y);
        }
        path.closePath();
        return path;
    }
}
```

### 6.3 Система частиц

```typescript
class Particle {
    x: number;
    y: number;
    vx: number;
    vy: number;
    life: number;
    maxLife: number;
    color: string;
    size: number;
}

class ParticleSystem {
    private particles: Particle[] = [];
    private ctx: CanvasRenderingContext2D;

    constructor(ctx: CanvasRenderingContext2D) {
        this.ctx = ctx;
    }

    emit(type: ParticleType, position: PixelCoord, count: number = 10): void {
        for (let i = 0; i < count; i++) {
            const particle = this.createParticle(type, position);
            this.particles.push(particle);
        }
    }

    private createParticle(type: ParticleType, position: PixelCoord): Particle {
        const angle = Math.random() * Math.PI * 2;
        const speed = Math.random() * 3 + 2;

        return {
            x: position.x,
            y: position.y,
            vx: Math.cos(angle) * speed,
            vy: Math.sin(angle) * speed,
            life: 1,
            maxLife: 30 + Math.random() * 20,
            color: this.getParticleColor(type),
            size: Math.random() * 4 + 2
        };
    }

    update(): void {
        for (let i = this.particles.length - 1; i >= 0; i--) {
            const p = this.particles[i];
            p.x += p.vx;
            p.y += p.vy;
            p.vy += 0.1; // Гравитация
            p.life--;

            if (p.life <= 0) {
                this.particles.splice(i, 1);
            }
        }
    }

    render(): void {
        for (const p of this.particles) {
            this.ctx.globalAlpha = p.life / p.maxLife;
            this.ctx.fillStyle = p.color;
            this.ctx.beginPath();
            this.ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
            this.ctx.fill();
        }
        this.ctx.globalAlpha = 1;
    }
}
```

---

## 7. Система сохранений

### 7.1 Структура сохранения

```typescript
interface GameSaveData {
    version: number;
    highScore: number;
    totalScore: number;
    sessionsPlayed: number;
    settings: GameSettings;
    lastSessionDate: string | null;
}

interface GameSettings {
    soundEnabled: boolean;
    musicEnabled: boolean;
    vibrationEnabled: boolean;
}

class SaveSystem {
    private readonly STORAGE_KEY = 'hex_block_game_save';
    private readonly PLAYGAMA_KEY = 'playgama_save';

    save(data: GameSaveData): void {
        const json = JSON.stringify(data);

        // PlayGama Storage (приоритет)
        if (window.PlayGamaSDK) {
            try {
                window.PlayGamaSDK.saveData(json);
                return;
            } catch (e) {
                console.warn('PlayGama save failed, falling back to localStorage');
            }
        }

        // LocalStorage (fallback)
        localStorage.setItem(this.STORAGE_KEY, json);
    }

    load(): GameSaveData | null {
        let json: string | null = null;

        // PlayGama Storage (приоритет)
        if (window.PlayGamaSDK) {
            try {
                json = window.PlayGamaSDK.loadData();
            } catch (e) {
                console.warn('PlayGama load failed, falling back to localStorage');
            }
        }

        // LocalStorage (fallback)
        if (!json) {
            json = localStorage.getItem(this.STORAGE_KEY);
        }

        if (!json) return null;

        try {
            return JSON.parse(json);
        } catch (e) {
            console.error('Failed to parse save data');
            return null;
        }
    }

    clear(): void {
        localStorage.removeItem(this.STORAGE_KEY);
        if (window.PlayGamaSDK) {
            window.PlayGamaSDK.clearData();
        }
    }
}
```

---

## 8. Интеграция аналитики

### 8.1 GameAnalytics

```typescript
class GameAnalyticsWrapper {
    private static initialized = false;
    private static buildVersion = CONFIG.version;

    static initialize(): void {
        if (this.initialized) return;

        if (window.GameAnalytics) {
            window.GameAnalytics.initialize({
                gameKey: CONFIG.gameAnalyticsKey,
                gameSecret: CONFIG.gameAnalyticsSecret
            });
            this.initialized = true;
        }
    }

    static trackEvent(eventName: string, params?: Record<string, unknown>): void {
        if (window.GameAnalytics) {
            window.GameAnalytics.addEventWithName(eventName, params);
        }
        console.log(`[Analytics] ${eventName}`, params);
    }

    // Gameplay events
    static trackGameStart(): void {
        this.trackEvent('game_start');
    }

    static trackGameOver(score: number, duration: number): void {
        this.trackEvent('game_over', { score, duration });
    }

    static trackFigurePlace(figureType: string, rotation: number): void {
        this.trackEvent('figure_place', { figure_type: figureType, rotation });
    }

    static trackLineDestroy(lineCount: number, scoreGained: number): void {
        this.trackEvent('line_destroy', { lines_count: lineCount, score_gained });
    }

    // Business events
    static trackAdRequest(placement: string): void {
        this.trackEvent('ad_request', { placement });
    }

    static trackAdCompleted(placement: string): void {
        this.trackEvent('ad_completed', { placement });
    }

    // Progression events
    static trackScoreMilestone(score: number): void {
        this.trackEvent('score_milestone', { score });
    }
}
```

---

## 9. Игровой цикл

### 9.1 GameLoop

```typescript
class GameLoop {
    private lastTime: number = 0;
    private deltaTime: number = 0;
    private isRunning: boolean = false;
    private animationFrameId: number | null = null;

    constructor(
        private update: (dt: number) => void,
        private render: () => void
    ) {}

    start(): void {
        if (this.isRunning) return;

        this.isRunning = true;
        this.lastTime = performance.now();
        this.animationFrameId = requestAnimationFrame(this.loop);
    }

    stop(): void {
        if (!this.isRunning) return;

        this.isRunning = false;
        if (this.animationFrameId) {
            cancelAnimationFrame(this.animationFrameId);
            this.animationFrameId = null;
        }
    }

    private loop = (currentTime: number): void => {
        if (!this.isRunning) return;

        this.deltaTime = currentTime - this.lastTime;
        this.lastTime = currentTime;

        // Ограничение deltaTime для избежания больших скачков
        const dt = Math.min(this.deltaTime, 50);

        this.update(dt);
        this.render();

        this.animationFrameId = requestAnimationFrame(this.loop);
    };
}
```

---

## 10. Конфигурация

### 10.1 Constants

```typescript
const CONFIG = {
    // Игровые параметры
    gridRadius: 3,              // Радиус для 5-клеточного диаметра
    cellSize: 40,               // Размер ячейки в пикселях
    hexScale: 0.95,             // Масштаб гекса внутри ячейки

    // Цветовая схема (dark theme)
    colors: {
        background: '#1A1A2E',
        grid: '#16213E',
        hover: '#0F3460',
        border: '#533483',
        ui: '#E94560'
    },

    // Очки
    pointsPerCell: 10,
    pointsPerLine: 100,

    // Комбо множители
    comboMultipliers: {
        1: 1,
        2: 1.5,
        3: 2,
        4: 3,
        5: 5
    },

    // Версия игры
    version: '1.0.0',

    // GameAnalytics ключи (заполняется из env)
    gameAnalyticsKey: '',
    gameAnalyticsSecret: ''
};
```

---

## 11. Тестирование

### 11.1 Структура тестов

```
tests/
├── unit/
│   ├── HexGrid.test.ts
│   ├── HexCoord.test.ts
│   ├── LineDetector.test.ts
│   ├── Figure.test.ts
│   ├── FigureGenerator.test.ts
│   └── ScoreManager.test.ts
├── integration/
│   ├── GameLoop.test.ts
│   ├── SaveSystem.test.ts
│   └── InputHandler.test.ts
└── e2e/
    └── game.test.ts
```

### 11.2 Пример unit-теста

```typescript
describe('HexCoord', () => {
    describe('axialToCube', () => {
        it('should convert axial to cube correctly', () => {
            const axial = { q: 1, r: 2 };
            const cube = HexCoord.axialToCube(axial);

            expect(cube.q).toBe(1);
            expect(cube.r).toBe(2);
            expect(cube.s).toBe(-3);
        });
    });

    describe('cubeDistance', () => {
        it('should calculate distance between two cells', () => {
            const a = { q: 0, r: 0, s: 0 };
            const b = { q: 2, r: -1, s: -1 };

            const distance = HexCoord.cubeDistance(a, b);
            expect(distance).toBe(2);
        });
    });
});
```

---

## 12. Сборка и деплой

### 12.1 Vite конфигурация

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
    build: {
        lib: {
            entry: 'src/main.ts',
            name: 'HexBlockGame',
            fileName: 'game'
        },
        rollupOptions: {
            output: {
                manualChunks: undefined
            }
        },
        minify: 'terser',
        terserOptions: {
            compress: {
                drop_console: true,
                drop_debugger: true
            }
        }
    },
    define: {
        'process.env': {}
    }
});
```

### 12.2 NPM скрипты

```json
{
    "scripts": {
        "dev": "vite",
        "build": "tsc && vite build",
        "preview": "vite preview",
        "test": "vitest",
        "lint": "eslint src --ext ts",
        "lint:fix": "eslint src --ext ts --fix"
    }
}
```

---

## 13. Чек-лист перед релизом

- [ ] Все unit-тесты проходят (>90% coverage)
- [ ] Интеграционные тесты проходят
- [ ] Производительность: 60 FPS стабильно
- [ ] Размер билда: < 30 KB (gzipped)
- [ ] Время загрузки: < 1.5 сек (3G)
- [ ] PlayGama интеграция работает
- [ ] GameAnalytics интеграция работает
- [ ] Сохранение/загрузка работает на всех платформах
- [ ] Touch events работают на мобильных
- [ ] Canvas адаптируется под размер экрана
- [ ] Нет утечек памяти
- [ ] ESLint/Prettier проверки проходят

---

**Документ подготовлен для команды разработки.**
