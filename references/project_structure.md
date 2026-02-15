# Project Structure

## Recommended Folder Layout

```
src/
├── dApp-kit.ts              # createDAppKit config + type registration
├── constants.ts             # Package IDs, object IDs, network config
├── main.tsx                 # Entry point with providers
├── App.tsx                  # Router + top-level layout
├── index.css                # Global styles
│
├── contracts/               # Codegen output (auto-generated)
│   └── game/
│       ├── game_module.ts   # Generated move call functions
│       └── utils/
│           └── index.ts     # MoveStruct helpers
│
├── hooks/                   # Custom React hooks
│   ├── useGameState.ts      # Read game session, world, grid
│   ├── usePlayerEntity.ts   # Read player entity state
│   ├── useGameActions.ts    # Execute game transactions
│   └── useTurnState.ts      # Track whose turn it is
│
├── components/              # React components
│   ├── layout/
│   │   ├── Header.tsx       # Wallet connect + network badge
│   │   └── GameLayout.tsx   # Main game layout wrapper
│   │
│   ├── game/
│   │   ├── GameBoard.tsx    # Grid / board renderer
│   │   ├── GameLobby.tsx    # Pre-game lobby (join, waiting)
│   │   ├── GameOver.tsx     # Win/loss screen
│   │   ├── PlayerHud.tsx    # Health, energy, status bars
│   │   └── ActionBar.tsx    # Turn action buttons
│   │
│   ├── entities/
│   │   ├── EntityCard.tsx   # Render an entity's stats
│   │   └── GridCell.tsx     # Single cell in the grid
│   │
│   └── ui/                  # Reusable UI primitives
│       ├── Button.tsx
│       ├── Modal.tsx
│       ├── ProgressBar.tsx
│       └── StatusBadge.tsx
│
├── lib/                     # Non-React utilities
│   ├── types.ts             # TypeScript types matching Move structs
│   ├── parsers.ts           # Parse on-chain object data → typed objects
│   └── gameLogic.ts         # Read-only calculations (display-only)
│
└── stores/                  # Client-side state (Zustand)
    └── uiStore.ts           # UI state: selected entity, camera, modals
```

## Key Principles

### 1. Contracts Are Read-Only Generated Code

Never edit files in `contracts/`. They are output from `@mysten/codegen` and will be overwritten.

### 2. Hooks Are the Bridge

Every on-chain read or write goes through a custom hook:
- `useGameState()` → reads World, Grid, GameSession
- `usePlayerEntity(entityId)` → reads a specific entity
- `useGameActions()` → returns functions like `move()`, `attack()`, `endTurn()`

### 3. Types Mirror Move Structs

Define TypeScript types that match your Move struct fields:

```typescript
// src/lib/types.ts
export interface GameSession {
  state: 'lobby' | 'active' | 'finished';
  players: string[];
  maxPlayers: number;
  winner: string | null;
  creator: string;
}

export interface PlayerEntity {
  id: string;
  owner: string;
  health: { current: number; max: number };
  energy: { current: number; max: number };
  position: { x: number; y: number };
}

export interface GridCell {
  x: number;
  y: number;
  occupant: string | null;  // entity ID
  terrain: string;
}
```

### 4. Parsers Convert Raw Data

On-chain data comes as generic objects — parse them into your types:

```typescript
// src/lib/parsers.ts
export function parseGameSession(fields: Record<string, any>): GameSession {
  return {
    state: parseState(fields.state),
    players: fields.players,
    maxPlayers: Number(fields.max_players),
    winner: fields.winner?.Some ?? null,
    creator: fields.creator,
  };
}
```

### 5. UI Store for Client-Only State

Use Zustand for state that doesn't exist on-chain:

```typescript
// src/stores/uiStore.ts
import { create } from 'zustand';

interface UIStore {
  selectedEntity: string | null;
  selectEntity: (id: string | null) => void;
  showActionModal: boolean;
  toggleActionModal: () => void;
}

export const useUIStore = create<UIStore>((set) => ({
  selectedEntity: null,
  selectEntity: (id) => set({ selectedEntity: id }),
  showActionModal: false,
  toggleActionModal: () => set((s) => ({ showActionModal: !s.showActionModal })),
}));
```
