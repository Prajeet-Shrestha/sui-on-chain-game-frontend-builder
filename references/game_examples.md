# Game Examples Reference

Curated patterns from the 8 existing game frontends. When building a new game, find the most similar example and use it as a reference.

---

## Game Catalog

| Game | Type | Grid? | Components | Key Patterns |
|------|------|-------|------------|--------------|
| **virus_game** | Puzzle | ✅ 5×5→12×12 | 7 | Multi-step PTB, Phaser renderer, local engine |
| **sokoban** | Puzzle | ✅ 6×6 | 5 | Local simulation, level data, directional input |
| **card_crawler** | Roguelike | ✅ 3×3 | 13 | Card system, animations, entity components |
| **tactics_ogre** | Tactics | ✅ variable | 9 | Multi-entity, turn-based, 6 hooks |
| **tetris_game** | Arcade | ✅ 10×20 | 3 | Real-time tick, keyboard input |
| **maze_game** | Puzzle | ✅ variable | 3 | Pathfinding, fog of war |
| **flappy_bird** | Arcade | ❌ | 3 | Physics loop, collision detection |
| **sandbox_game** | Sandbox | ✅ variable | 3 | Entity editor, creative mode |

---

## Pattern Index

### Grid/Board Rendering
**Best examples:** virus_game, sokoban, card_crawler

The most common pattern — render a 2D grid from on-chain state:

```
Game state (on-chain) → useGameSession (poll) → parsers → GameBoard component
```

- **CSS Grid**: sokoban, card_crawler, maze_game — use CSS `grid-template-columns`
- **Phaser Canvas**: virus_game — for animations, particles, camera effects
- **Simple divs**: tetris_game — absolute positioning

### Multi-Step PTB (Programmable Transaction Block)
**Best example:** virus_game

Chain multiple Move calls in one transaction:
```
start_level() → choose_color() × N → share_session()
```

All calls share the same transaction, so they're atomic.

### Local Simulation
**Best examples:** virus_game (`lib/localEngine.ts`), sokoban (`utils/`)

Run game logic **client-side** to preview moves before submitting on-chain:

```
User input → local simulation (instant) → show preview → submit PTB → confirm on-chain
```

### Level Selection
**Best examples:** virus_game, sokoban, card_crawler

Pattern: Level metadata in `constants.ts`, level select component, start game action:

```typescript
export const LEVELS = [
    { id: 1, name: 'Easy', ... },
    { id: 2, name: 'Medium', ... },
];
```

### Card-Based UI
**Best example:** card_crawler

Uses `data/cardLookup.ts` for card definitions, `lib/parsers.ts` for card state parsing, and `components/` for Card, Hand, Board rendering.

### Real-Time / Physics
**Best examples:** flappy_bird, tetris_game

Uses `requestAnimationFrame` or `setInterval` for client-side animation loop. On-chain state is snapshot-based, not real-time.

### Animation-Heavy UI
**Best example:** card_crawler (`animations.css`, 13 components)

Separate `animations.css` file with keyframe animations, screen transitions (`useScreenTransition` hook).

### Multi-Entity / Turn-Based
**Best example:** tactics_ogre (6 hooks, 9 components)

Most complex frontend — multiple entity types, turn management, action selection:

```
hooks/
├── useGameActions.ts
├── useGameSession.ts
├── useGrid.ts
├── useEntityComponents.ts
├── useTurnState.ts
└── usePlayerEntity.ts
```

---

## Directory Structures by Complexity

### Simple (flappy_bird, maze_game, sandbox_game, tetris_game)
```
src/
├── App.tsx, main.tsx, dApp-kit.ts, constants.ts, index.css
├── components/ (3 files)
├── hooks/ (1 file)
├── lib/ (2-5 files)
└── stores/ (1 file)
```

### Medium (sokoban, virus_game)
```
src/
├── App.tsx, main.tsx, dApp-kit.ts, constants.ts, index.css
├── components/ (5-7 files)
├── hooks/ (2 files)
├── lib/ (1-5 files)
├── stores/ (1 file)
└── utils/ or assets/ (optional)
```

### Complex (card_crawler, tactics_ogre)
```
src/
├── App.tsx, main.tsx, dApp-kit.ts, constants.ts, index.css
├── animations.css (optional)
├── components/ (9-13 files)
├── hooks/ (4-6 files)
├── lib/ (3-4 files)
├── data/ (optional)
├── stores/ (1 file)
└── images/ or assets/ (optional)
```
