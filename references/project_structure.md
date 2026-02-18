# Project Structure

## Recommended Folder Layout

Based on all 8 existing game frontends. Adjust component count to game complexity.

```
frontend/
├── index.html               # Entry HTML (Google Fonts, meta tags)
├── package.json             # Dependencies (proven versions)
├── vite.config.ts           # Vite config with base path
├── tsconfig.json            # References tsconfig.app.json
├── tsconfig.app.json        # TypeScript compiler options
│
└── src/
    ├── main.tsx              # Entry point — providers (DO NOT MODIFY)
    ├── dApp-kit.ts           # createDAppKit config + type registration
    ├── constants.ts          # Package IDs, object IDs, states, error codes
    ├── App.tsx               # Top-level layout + game state routing
    ├── index.css             # All styles (premium, dark mode, animations)
    │
    ├── hooks/                # Custom React hooks
    │   ├── useGameActions.ts # Execute game transactions (PTB builders)
    │   └── useGameSession.ts # Read game state via React Query polling
    │   # Optional: usePlayerEntity.ts, useTurnState.ts, useGrid.ts
    │
    ├── components/           # React components (game-specific)
    │   ├── Header.tsx        # Wallet connect + game title
    │   ├── GameBoard.tsx     # Grid / board renderer
    │   └── GameOver.tsx      # Win/loss screen
    │   # Optional: LevelSelect.tsx, ActionBar.tsx, PlayerHud.tsx, etc.
    │
    ├── lib/                  # Non-React utilities
    │   ├── suiClient.ts      # SuiJsonRpcClient for data reads
    │   ├── types.ts          # TypeScript interfaces matching Move structs
    │   └── parsers.ts        # Parse on-chain JSON-RPC → typed objects
    │   # Optional: localEngine.ts, cardLookup.ts, etc.
    │
    ├── stores/               # Client-side state (Zustand)
    │   └── uiStore.ts        # UI state: sessionId, isPending, error
    │
    └── assets/               # Optional: images, sprites, sounds
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
