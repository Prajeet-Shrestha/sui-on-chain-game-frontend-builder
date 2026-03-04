# Scaffolding Protocol

**Mandatory step-by-step process** for creating a new game frontend. Follow in order.

---

## Phase 1: Contract Analysis

**Before writing any frontend code, read the Move contract.**

1. Open `sources/game.move` (or the main module file)
2. Extract the following (see [contract_integration.md](contract_integration.md) for details):
   - All `public struct` declarations → will become `lib/types.ts`
   - All `const` values (states, error codes) → will become `constants.ts`
   - All `public fun` / `entry fun` signatures → will become `hooks/useGameActions.ts`
   - All shared objects created in `init()` → will become WORLD_ID, etc. in `constants.ts`

---

## Phase 2: Scaffold Boilerplate

Create the `frontend/` directory and drop in these **unchanged templates**:

| Order | File | Template |
|-------|------|----------|
| 1 | `package.json` | [package_json.md](../templates/package_json.md) — replace `__GAME_SLUG__` |
| 2 | `vite.config.ts` | [vite_config.ts.md](../templates/vite_config.ts.md) |
| 3 | `index.html` | [index_html.md](../templates/index_html.md) — replace `__GAME_TITLE__` |
| 4 | `tsconfig.json` | See below |
| 5 | `src/main.tsx` | [main.tsx.md](../templates/main.tsx.md) |
| 6 | `src/dApp-kit.ts` | [dapp_kit.ts.md](../templates/dapp_kit.ts.md) |
| 7 | `src/lib/suiClient.ts` | [sui_client.ts.md](../templates/sui_client.ts.md) |
| 8 | `src/stores/uiStore.ts` | [ui_store.ts.md](../templates/ui_store.ts.md) |

### tsconfig.json

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

### tsconfig.app.json

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "types": ["vite/client"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

### tsconfig.node.json

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "ESNext",
    "types": ["node"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["vite.config.ts"]
}
```

---

## Phase 3: Generate Contract Integration Layer

These files are **game-specific** — generate them from the contract analysis:

| Order | File | Source |
|-------|------|--------|
| 9 | `src/constants.ts` | Contract consts + deployed IDs |
| 10 | `src/lib/types.ts` | Contract structs → TS interfaces |
| 11 | `src/lib/parsers.ts` | Field mapping functions |

Follow [contract_integration.md](contract_integration.md) for the conversion rules.

---

## Phase 4: Generate Hooks

| Order | File | Source |
|-------|------|--------|
| 12 | `src/hooks/useGameActions.ts` | Contract public functions → PTB builders |
| 13 | `src/hooks/useGameSession.ts` | Query pattern for main game object |

Use templates from [use_game_actions.ts.md](../templates/use_game_actions.ts.md) and [use_game_session.ts.md](../templates/use_game_session.ts.md) as starting points, then customize for your game's specific functions.

---

## Phase 5: Build Game UI

Now create game-specific components:

| Order | File | Purpose |
|-------|------|---------|
| 14 | `src/App.tsx` | Main app — wallet connect + game state routing |
| 15 | `src/components/GameBoard.tsx` | Primary game view |
| 16 | `src/components/Header.tsx` | ConnectButton + game title |
| 17 | `src/components/GameOver.tsx` | Win/loss screen |
| 18 | `src/index.css` | All styles (premium, dark mode, animations) |

### App.tsx Pattern

```tsx
import { ConnectButton } from '@mysten/dapp-kit-react/ui';
import { useCurrentAccount } from '@mysten/dapp-kit-react';
import { useUIStore } from './stores/uiStore';
import { useGameSession } from './hooks/useGameSession';
// import game components...

function App() {
    const account = useCurrentAccount();
    const { sessionId } = useUIStore();
    const { data: session } = useGameSession(sessionId);

    return (
        <div className="app">
            <header>
                <h1>Game Title</h1>
                <ConnectButton />
            </header>
            <main>
                {!account ? (
                    <div className="connect-prompt">Connect wallet to play</div>
                ) : !session ? (
                    <LevelSelect />
                ) : session.state === 'active' ? (
                    <GameBoard session={session} />
                ) : (
                    <GameOver session={session} />
                )}
            </main>
        </div>
    );
}
```

---

## Phase 6: Install & Verify

```bash
cd frontend
npm install
npm run dev
# Open http://localhost:5173
```

Verify:
1. Page loads without console errors
2. ConnectButton renders and wallet connects
3. Game state reads from chain (or shows loading)
4. Game actions submit transactions successfully

---

## File Creation Order Summary

```
1.  package.json          ← template
2.  vite.config.ts        ← template (replace __GAME_SLUG__)
3.  index.html            ← template (replace __GAME_TITLE__)
4.  tsconfig.json         ← template
5.  tsconfig.app.json     ← template
6.  tsconfig.node.json    ← template
7.  src/main.tsx           ← template (unchanged)
8.  src/dApp-kit.ts        ← template (unchanged)
9.  src/lib/suiClient.ts   ← template (unchanged)
10. src/stores/uiStore.ts  ← template (may extend)
11. src/constants.ts       ← generated from contract
12. src/lib/types.ts       ← generated from contract structs
13. src/lib/parsers.ts     ← generated from contract structs
14. src/hooks/useGameActions.ts  ← generated from contract fns
15. src/hooks/useGameSession.ts  ← generated from contract structs
16. src/App.tsx            ← game-specific
17. src/components/*.tsx   ← game-specific
18. src/index.css          ← game-specific (premium design)
```
