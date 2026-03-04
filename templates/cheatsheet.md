# Templates Cheatsheet

> Distilled from 10 template files. Copy-paste ready scaffolding for Sui game frontends.

---

## 1. Project Scaffold

### package.json

```json
{
    "name": "__GAME_SLUG__-frontend",
    "private": true,
    "version": "0.0.0",
    "type": "module",
    "scripts": {
        "dev": "vite",
        "build": "tsc -b && vite build",
        "preview": "vite preview"
    },
    "dependencies": {
        "@mysten/dapp-kit-react": "^2.0.0",
        "@mysten/sui": "^2.4.0",
        "@tanstack/react-query": "^5.90.21",
        "react": "^19.2.0",
        "react-dom": "^19.2.0",
        "zustand": "^5.0.11"
    },
    "devDependencies": {
        "@types/react": "^19.2.7",
        "@types/react-dom": "^19.2.3",
        "@vitejs/plugin-react": "^5.1.1",
        "typescript": "~5.9.3",
        "vite": "^7.3.1"
    }
}
```

**Optional deps** — add only when needed:

| Package | Version | When |
|---------|---------|------|
| `phaser` | `^3.90.0` | Canvas 2D games |
| `framer-motion` | `^11.x` | Smooth animations |
| `lucide-react` | `^0.x` | Icons |
| `@eslint/js` | `^9.39.1` | Linting (add with eslint config) |

### vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    base: '/__GAME_SLUG__/',
    plugins: [react()],
    resolve: { alias: { '@': '/src' } },
    build: { target: 'esnext' }, // gRPC needs modern JS
});
```

> `base` must match the game slug from `example.json`.

<details>
<summary>Optional: Chunk splitting for large games</summary>

```typescript
build: {
    rollupOptions: {
        output: {
            manualChunks(id) {
                if (id.includes('node_modules/react') || id.includes('node_modules/react-dom')) return 'vendor-react';
                if (id.includes('node_modules/@mysten')) return 'vendor-sui';
                if (id.includes('node_modules/@tanstack')) return 'vendor-query';
            },
        },
    },
},
```
</details>

### tsconfig.json

```json
{ "files": [], "references": [{ "path": "./tsconfig.app.json" }, { "path": "./tsconfig.node.json" }] }
```

### tsconfig.app.json

```json
{
    "compilerOptions": {
        "target": "ES2022", "lib": ["ES2022", "DOM", "DOM.Iterable"],
        "module": "ESNext", "moduleResolution": "bundler",
        "jsx": "react-jsx", "strict": true, "noEmit": true,
        "skipLibCheck": true, "verbatimModuleSyntax": true,
        "allowImportingTsExtensions": true, "types": ["vite/client"]
    },
    "include": ["src"]
}
```

> `"types": ["vite/client"]` handles CSS module imports — no `vite-env.d.ts` needed.

### tsconfig.node.json

```json
{
    "compilerOptions": {
        "target": "ES2023", "lib": ["ES2023"],
        "module": "ESNext", "moduleResolution": "bundler",
        "strict": true, "noEmit": true, "skipLibCheck": true,
        "verbatimModuleSyntax": true
    },
    "include": ["vite.config.ts"]
}
```

### index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <!-- Optional: Google Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet" />
    <title>__GAME_TITLE__</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**Font reference:**

| Style | Font | Games |
|-------|------|-------|
| Medieval / Fantasy | Cinzel Decorative, MedievalSharp | card_crawler |
| Pixel / Retro | Press Start 2P | tactics_ogre |
| Futuristic / Sci-fi | Orbitron, Space Mono | tetris_game |
| Modern / Clean | (CSS-only, no custom fonts) | virus_game, sokoban |

---

## 2. Core Wiring (identical across all games)

### src/dApp-kit.ts — dApp Kit Config

```typescript
import { createDAppKit } from '@mysten/dapp-kit-react';
import { SuiGrpcClient } from '@mysten/sui/grpc';

const GRPC_URLS: Record<string, string> = {
    testnet: 'https://fullnode.testnet.sui.io:443',
    devnet: 'https://fullnode.devnet.sui.io:443',
};

export const dAppKit = createDAppKit({
    networks: ['testnet', 'devnet'],
    defaultNetwork: 'testnet',
    enableBurnerWallet: import.meta.env.DEV,
    createClient(network) {
        return new SuiGrpcClient({
            network,
            baseUrl: GRPC_URLS[network] ?? GRPC_URLS.testnet,
        });
    },
});

// REQUIRED — enables typed hooks
declare module '@mysten/dapp-kit-react' {
    interface Register { dAppKit: typeof dAppKit; }
}
```

> Without the `declare module` block, hooks like `useCurrentNetwork()` won't infer network types.

<details>
<summary>Minimal variant (no explicit URLs)</summary>

```typescript
export const dAppKit = createDAppKit({
    networks: ['testnet', 'devnet'],
    defaultNetwork: 'testnet',
    enableBurnerWallet: import.meta.env.DEV,
    createClient(network) {
        return new SuiGrpcClient({ network });
    },
});
```
Used by: tetris_game, sandbox_game, card_crawler.
</details>

### src/main.tsx — Entry Point

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { DAppKitProvider } from '@mysten/dapp-kit-react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { dAppKit } from './dApp-kit';
import App from './App';
import './index.css';

const queryClient = new QueryClient({
    defaultOptions: { queries: { staleTime: 2_000, refetchOnWindowFocus: false } },
});

createRoot(document.getElementById('root')!).render(
    <StrictMode>
        <QueryClientProvider client={queryClient}>
            <DAppKitProvider dAppKit={dAppKit}>
                <App />
            </DAppKitProvider>
        </QueryClientProvider>
    </StrictMode>,
);
```

**Provider order matters:** `QueryClientProvider` → `DAppKitProvider` → `App`.

### src/lib/suiClient.ts — Standalone gRPC Client

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';

export const suiClient = new SuiGrpcClient({
    network: 'testnet',
    baseUrl: 'https://fullnode.testnet.sui.io:443',
});
```

> **Why separate?** This standalone `SuiGrpcClient` reads data before any wallet connects (e.g., displaying game state on page load). It uses the same Core API as the dApp Kit client.

### src/App.tsx — Game Shell

```tsx
import { ConnectButton } from '@mysten/dapp-kit-react/ui';
import { useCurrentAccount } from '@mysten/dapp-kit-react';

function App() {
    const account = useCurrentAccount();
    if (!account) {
        return (
            <div className="connect-screen">
                <h1>__GAME_TITLE__</h1>
                <ConnectButton />
            </div>
        );
    }
    return (
        <div className="app">
            <header><ConnectButton /></header>
            {/* Game UI goes here */}
        </div>
    );
}
export default App;
```

> **Import rule:** `ConnectButton`/`ConnectModal` → from `@mysten/dapp-kit-react/ui`. All hooks/providers → from `@mysten/dapp-kit-react` (root).

---

## 3. Game State Layer

### src/constants.ts — Contract IDs & Mappings

```typescript
// ── Deployed contract IDs (update after `sui client publish`) ──
export const PACKAGE_ID = '0x__REPLACE_WITH_PUBLISHED_PACKAGE_ID__';
export const WORLD_ID   = '0x__REPLACE_WITH_WORLD_OBJECT_ID__';

// ── Game states (must match Move `const STATE_*` values) ──
export const STATE_LOBBY    = 0;
export const STATE_ACTIVE   = 1;
export const STATE_FINISHED = 2;

// ── Error codes (must match Move `const E_*: u64` abort codes) ──
export const GAME_ERROR_MAP: Record<number, string> = {
    100: 'Invalid game state',
    101: 'Not the player',
    // Map every abort code to a human-readable message
};
```

**How to extract values:**
- `PACKAGE_ID` / `WORLD_ID` → from `sui client publish` output
- States → search `.move` for `const STATE_*`
- Errors → search for `const E*: u64` patterns

### src/stores/uiStore.ts — Zustand UI Store

```typescript
import { create } from 'zustand';

interface UIStore {
    sessionId: string | null;
    setSessionId: (id: string | null) => void;
    isPending: boolean;
    setIsPending: (pending: boolean) => void;
    error: string | null;
    setError: (error: string | null) => void;
}

export const useUIStore = create<UIStore>((set) => ({
    sessionId: null,
    setSessionId: (id) => set({ sessionId: id }),
    isPending: false,
    setIsPending: (pending) => set({ isPending: pending }),
    error: null,
    setError: (error) => set({ error }),
}));
```

**Extend per game:** add `selectedLevel`, `showModal`, `selectedEntity`, etc.

---

## 4. Hooks

### src/hooks/useGameActions.ts — Transaction Execution

Core pattern: build PTB → sign & execute → wait for finality → manage pending/error state.

> **Client distinction:** `useCurrentClient()` returns the **gRPC client** from dAppKit (wallet-connected). The standalone `suiClient` (also gRPC) is for pre-wallet reads like displaying game state on page load.

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { PACKAGE_ID, WORLD_ID, GAME_ERROR_MAP } from '../constants';
import { useUIStore } from '../stores/uiStore';

function parseTransactionError(failure: { error: string }): Error {
    const errorStr = typeof failure.error === 'string'
        ? failure.error : JSON.stringify(failure.error);
    const match = errorStr.match(/MoveAbort.*?(\d+)\)?$/);
    if (match) {
        const code = Number(match[1]);
        return new Error(GAME_ERROR_MAP[code] ?? `Transaction failed (code ${code})`);
    }
    return new Error(errorStr || 'Transaction failed');
}

export function useGameActions() {
    const dAppKit = useDAppKit();
    const client = useCurrentClient();
    const { setIsPending, setError } = useUIStore();

    async function execute(tx: Transaction) {
        setIsPending(true);
        setError(null);
        try {
            const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
            if (result.FailedTransaction) {
                throw parseTransactionError(result.FailedTransaction as any);
            }
            await client.core.waitForTransaction({
                digest: result.Transaction.digest,
                include: { effects: true },
            });
            return result.Transaction;
        } catch (err: any) {
            const message = err instanceof Error ? err.message : 'Transaction failed';
            setError(message);
            throw err;
        } finally {
            setIsPending(false);
        }
    }

    return {
        // Define game-specific actions here, e.g.:
        // startGame: (level: number) => {
        //     const tx = new Transaction();
        //     tx.moveCall({ package: PACKAGE_ID, module: 'game', function: 'start_game',
        //         arguments: [tx.object(WORLD_ID), tx.pure.u64(level)] });
        //     return execute(tx);
        // },
    };
}
```

#### Key Patterns

| Pattern | Use Case | Example Game |
|---------|----------|-------------|
| **Discover created objects** | tx creates new shared objects → extract from events/objectChanges | sokoban, tactics_ogre, flappy_bird |
| **Refetch React Query** | invalidate caches post-tx via `queryClient.refetchQueries` | tactics_ogre |
| **Multi-step PTB** | chain multiple `tx.moveCall` in one atomic transaction | virus_game |
| **Pass vectors** | `tx.pure.vector('u8', dirs)` or `tx.makeMoveVec({ type, elements })` | sokoban |
| **Auto-fund faucet** | check balance → `requestSuiFromFaucetV2` if low | flappy_bird |

<details>
<summary>Pattern: Multi-step PTB (chain calls atomically)</summary>

```typescript
const [session] = tx.moveCall({ ...startLevel });
for (const color of colors) {
    tx.moveCall({ ...chooseColor, arguments: [session, tx.pure.u8(color)] });
}
tx.moveCall({ ...shareSession, arguments: [session] });
```
</details>

<details>
<summary>Pattern: Pass vectors</summary>

```typescript
// vector<u8> from array
tx.pure.vector('u8', directions),

// Vector of objects with explicit type
const boxObjects = boxEntityIds.map(id => tx.object(id));
const boxVec = tx.makeMoveVec({
    type: `${ENTITY_PACKAGE_ID}::entity::Entity`,
    elements: boxObjects,
});
```
</details>

<details>
<summary>Pattern: Discover created objects from tx result</summary>

```typescript
const txResponse = await client.core.getTransaction({
    digest: result.Transaction.digest,
    include: { effects: true, events: true, objectChanges: true },
});
// Preferred: find by event type
const event = txResponse.Transaction?.events?.find(
    (e: any) => e.type?.includes('::game::LevelStarted')
);
const sessionId = event?.parsedJson?.session_id;
// Fallback: find by objectChanges
const created = txResponse.Transaction?.effects?.changedObjects?.filter(
    (c: any) => c.idOperation === 'Created'
);
const sessionObj = created?.find((c: any) => c.objectType?.includes('GameSession'));
```
</details>

<details>
<summary>Pattern: Refetch React Query caches post-transaction</summary>

```typescript
import { useQueryClient } from '@tanstack/react-query';
const queryClient = useQueryClient();

async function executeAndRefresh(tx: Transaction, extraKeys: string[] = []) {
    const result = await execute(tx);
    const keys = ['gameSession', 'grid', 'entities', ...extraKeys];
    await Promise.all(keys.map(k => queryClient.refetchQueries({ queryKey: [k] })));
    return result;
}
```
</details>

<details>
<summary>Pattern: Auto-fund with faucet</summary>

```typescript
import { requestSuiFromFaucetV2, getFaucetHost } from '@mysten/sui/faucet';

async function ensureGas(address: string, client: SuiGrpcClient) {
    const balance = await client.core.getBalance({ owner: address });
    if (BigInt(balance?.totalBalance ?? '0') < 10_000_000n) {
        await requestSuiFromFaucetV2({ host: getFaucetHost('testnet'), recipient: address });
        await new Promise(r => setTimeout(r, 1500));
    }
}
```
</details>

### src/hooks/useGameSession.ts — On-Chain State Polling

```typescript
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { parseGameSession } from '../lib/parsers';
import type { GameSession } from '../lib/types';

export function useGameSession(sessionId: string | null) {
    const client = useCurrentClient();
    return useQuery<GameSession>({
        queryKey: ['gameSession', sessionId],
        queryFn: async () => {
            const { object } = await client.core.getObject({
                objectId: sessionId!,
                include: { json: true },
            });
            if (!object?.json) {
                throw new Error('GameSession not found');
            }
            return parseGameSession(
                object.objectId,
                object.json as Record<string, any>,
            );
        },
        enabled: !!sessionId,
        refetchInterval: 2_000,
    });
}
```

**Variations:**

<details>
<summary>Composite state (sokoban, tactics_ogre)</summary>

```typescript
export function useGameState(sessionId: string | null) {
    const session = useGameSession(sessionId);
    const grid = useGrid(session.data?.gridId ?? null);
    const entities = useEntities(session.data?.entityIds ?? []);
    return { session, grid, entities };
}
```
</details>

<details>
<summary>Owned objects (card_crawler)</summary>

```typescript
import { useCurrentAccount, useCurrentClient } from '@mysten/dapp-kit-react';

export function usePlayerGames() {
    const account = useCurrentAccount();
    const client = useCurrentClient();
    return useQuery({
        queryKey: ['playerGames', account?.address],
        queryFn: async () => {
            const res = await client.core.listOwnedObjects({
                owner: account!.address,
                filter: { StructType: `${PACKAGE_ID}::game::GameSession` },
                include: { json: true },
            });
            return res.objects.map(obj =>
                parseGameSession(obj.objectId, obj.content!.fields as any)
            );
        },
        enabled: !!account,
    });
}
```
</details>

---

## 5. Placeholder Tokens

Replace these before shipping:

| Token | Replace With |
|-------|-------------|
| `__GAME_SLUG__` | Game slug, e.g. `virus-game`, `card-crawler` |
| `__GAME_TITLE__` | Display name, e.g. "Virus Game", "Card Crawler" |
| `__REPLACE_WITH_PUBLISHED_PACKAGE_ID__` | `packageId` from `sui client publish` output |
| `__REPLACE_WITH_WORLD_OBJECT_ID__` | `World` shared object ID from publish output |

---

## 6. File Map

| File | Path | Identical? | Key Context |
|------|------|-----------|-------------|
| package.json | `./` | ✅ versions locked | Proven deps across 8 games |
| tsconfig.json | `./` | ✅ identical | References `tsconfig.app.json` + `tsconfig.node.json` |
| tsconfig.app.json | `./` | ✅ identical | `"types": ["vite/client"]` handles CSS imports |
| tsconfig.node.json | `./` | ✅ identical | For `vite.config.ts` only |
| vite.config.ts | `./` | ≈ `base` varies | `target: 'esnext'` required for gRPC |
| index.html | `./` | ≈ fonts vary | Add Google Fonts in `<head>` per theme |
| main.tsx | `src/` | ✅ identical | Provider nesting order is critical |
| App.tsx | `src/` | Game-specific | `ConnectButton` from `/ui`, game phase routing |
| dApp-kit.ts | `src/` | ≈ URLs optional | `declare module` block is mandatory |
| suiClient.ts | `src/lib/` | ✅ identical | Standalone gRPC client for pre-wallet reads |
| constants.ts | `src/` | Game-specific | All values must match Move contract |
| uiStore.ts | `src/stores/` | Core identical | Extend with game-specific fields |
| useGameActions.ts | `src/hooks/` | Core identical | Actions are game-specific, wrapper is shared |
| useGameSession.ts | `src/hooks/` | Core identical | Parser + types are game-specific |
