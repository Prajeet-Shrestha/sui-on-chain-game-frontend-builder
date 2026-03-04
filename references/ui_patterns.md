# UI Patterns & Performance

## Game Phase Views

Structure the app around game states:

```tsx
// src/App.tsx
import { ConnectButton } from '@mysten/dapp-kit-react/ui';
import { useCurrentAccount } from '@mysten/dapp-kit-react';
import { useGameState } from './hooks/useGameState';

function App() {
  const account = useCurrentAccount();
  const { data: game, isLoading } = useGameState();

  return (
    <div className="app">
      <header>
        <h1>My Game</h1>
        <ConnectButton />
      </header>

      <main>
        {!account ? (
          <WelcomeScreen />
        ) : isLoading ? (
          <LoadingScreen />
        ) : !game ? (
          <ErrorScreen />
        ) : (
          <GameView game={game} />
        )}
      </main>
    </div>
  );
}

function GameView({ game }: { game: GameSession }) {
  switch (game.state) {
    case 'lobby': return <GameLobby game={game} />;
    case 'active': return <GameActive game={game} />;
    case 'finished': return <GameOver game={game} />;
  }
}
```

---

## Grid / Board Renderer

### CSS Grid for Game Boards

```tsx
// src/components/game/GameBoard.tsx
import './GameBoard.css';

interface GridCellData {
  x: number;
  y: number;
  occupant: string | null;
  terrain: string;
}

function GameBoard({ cells, width, height, onCellClick }: {
  cells: GridCellData[];
  width: number;
  height: number;
  onCellClick: (x: number, y: number) => void;
}) {
  return (
    <div
      className="game-grid"
      style={{
        gridTemplateColumns: `repeat(${width}, 1fr)`,
        gridTemplateRows: `repeat(${height}, 1fr)`,
      }}
    >
      {cells.map((cell) => (
        <GridCell
          key={`${cell.x}-${cell.y}`}
          cell={cell}
          onClick={() => onCellClick(cell.x, cell.y)}
        />
      ))}
    </div>
  );
}
```

```css
/* src/components/game/GameBoard.css */
.game-grid {
  display: grid;
  gap: 2px;
  aspect-ratio: 1;
  max-width: 600px;
  margin: 0 auto;
}

.grid-cell {
  aspect-ratio: 1;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.grid-cell:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

.grid-cell--occupied {
  background-color: rgba(59, 130, 246, 0.3);
}

.grid-cell--selected {
  outline: 2px solid #3b82f6;
  outline-offset: -2px;
}

.grid-cell--reachable {
  background-color: rgba(34, 197, 94, 0.2);
}
```

---

## Player HUD

```tsx
// src/components/game/PlayerHud.tsx
function PlayerHud({ entity }: { entity: PlayerEntity }) {
  return (
    <div className="player-hud">
      <div className="stat-row">
        <label>HP</label>
        <ProgressBar
          current={entity.health.current}
          max={entity.health.max}
          color="red"
        />
      </div>
      <div className="stat-row">
        <label>Energy</label>
        <ProgressBar
          current={entity.energy.current}
          max={entity.energy.max}
          color="blue"
        />
      </div>
      <div className="position">
        Position: ({entity.position.x}, {entity.position.y})
      </div>
    </div>
  );
}

function ProgressBar({ current, max, color }: {
  current: number;
  max: number;
  color: string;
}) {
  const pct = Math.round((current / max) * 100);
  return (
    <div className="progress-bar">
      <div
        className="progress-bar__fill"
        style={{
          width: `${pct}%`,
          backgroundColor: `var(--color-${color})`,
        }}
      />
      <span className="progress-bar__text">{current}/{max}</span>
    </div>
  );
}
```

---

## Action Button with Loading State

```tsx
function ActionButton({ label, onClick, disabled }: {
  label: string;
  onClick: () => Promise<void>;
  disabled?: boolean;
}) {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleClick() {
    setIsPending(true);
    setError(null);
    try {
      await onClick();
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed');
    } finally {
      setIsPending(false);
    }
  }

  return (
    <div>
      <button onClick={handleClick} disabled={disabled || isPending}>
        {isPending ? `${label}...` : label}
      </button>
      {error && <p className="error-text">{error}</p>}
    </div>
  );
}
```

---

## Performance Tips

### 1. Memoize Expensive Renders

Grid cells re-render frequently during polling. Memoize them:

```tsx
const GridCell = React.memo(function GridCell({ cell, onClick }: {
  cell: GridCellData;
  onClick: () => void;
}) {
  return (
    <div
      className={`grid-cell ${cell.occupant ? 'grid-cell--occupied' : ''}`}
      onClick={onClick}
    >
      {cell.occupant && <EntityIcon entityId={cell.occupant} />}
    </div>
  );
});
```

### 2. Separate Queries per Object

Don't put everything in one giant query — split by update frequency:

```typescript
// Fast-changing (poll every 2s)
useQuery({ queryKey: ['turnState'], refetchInterval: 2_000 });

// Medium (poll every 3s)
useQuery({ queryKey: ['grid'], refetchInterval: 3_000 });

// Slow-changing (poll every 5s)
useQuery({ queryKey: ['gameSession'], refetchInterval: 5_000 });
```

### 3. Avoid Re-renders During Wallet-Only Changes

Don't use `useDAppKit()` in deeply nested components — it re-renders on any store change. Instead:

```tsx
// ❌ Bad — re-renders on every store change
function GridCell() {
  const dAppKit = useDAppKit();
  // ...
}

// ✅ Good — only gets what it needs
function GridCell() {
  const client = useCurrentClient(); // Only re-renders on network change
  // ...
}

// ✅ Even better — lift action hooks to parent
function GameBoard() {
  const { moveEntity } = useGameActions(); // one place
  return cells.map(cell => (
    <GridCell key={cell.id} onMove={(x, y) => moveEntity(cell.entityId, x, y)} />
  ));
}
```

### 4. Use CSS Transitions, Not JS

For animations (entity movement, health bar changes), prefer CSS transitions:

```css
.entity-sprite {
  transition: transform 0.3s ease-out;
}

.progress-bar__fill {
  transition: width 0.3s ease;
}
```

### 5. Preload Game Data

Fetch essential objects on mount, before the user interacts:

```typescript
// In App.tsx or a layout component
function GameLoader({ children }: { children: React.ReactNode }) {
  const gameState = useGameState();
  const grid = useGrid();
  const turnState = useTurnState();

  const isReady = gameState.data && grid.data && turnState.data;

  if (!isReady) return <LoadingScreen />;
  return <>{children}</>;
}
```
