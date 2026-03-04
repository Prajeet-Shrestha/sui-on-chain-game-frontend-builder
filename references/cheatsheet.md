# Sui Game Frontend — Cheatsheet

> Consolidated from 15 reference files for the `sui-on-chain-game-frontend-builder` skill.

---

## 1. Setup & Project Structure

### Install

```bash
npm create vite@latest my-game -- --template react-ts
cd my-game
npm i @mysten/dapp-kit-react @mysten/sui @tanstack/react-query
# Optional: npm i zustand framer-motion lucide-react
```

### Folder Layout

```
frontend/src/
├── main.tsx              # Providers (QueryClient + DAppKitProvider)
├── dApp-kit.ts           # createDAppKit + type registration
├── constants.ts          # Package IDs, object IDs, states, error codes
├── App.tsx               # Layout + game state routing
├── index.css             # All styles (dark mode, animations)
├── hooks/                # useGameActions.ts, useGameSession.ts
├── components/           # Header.tsx, GameBoard.tsx, GameOver.tsx
├── lib/                  # suiClient.ts, types.ts, parsers.ts
└── stores/               # uiStore.ts (Zustand)
```

### dApp Kit Config (`src/dApp-kit.ts`)

```typescript
import { createDAppKit } from '@mysten/dapp-kit-react';
import { SuiGrpcClient } from '@mysten/sui/grpc';

export const dAppKit = createDAppKit({
  networks: ['testnet', 'devnet'],
  defaultNetwork: 'testnet',
  enableBurnerWallet: import.meta.env.DEV,
  createClient(network) {
    return new SuiGrpcClient({
      network,
      baseUrl: `https://fullnode.${network}.sui.io:443`,
    });
  },
});

// REQUIRED: type registration for hook inference
declare module '@mysten/dapp-kit-react' {
  interface Register { dAppKit: typeof dAppKit; }
}
```

#### `createDAppKit` Options

| Option | Default | Description |
|--------|---------|-------------|
| `networks` | (required) | Array of network name strings |
| `createClient(network)` | (required) | Factory returning a client |
| `defaultNetwork` | `networks[0]` | Initial active network |
| `autoConnect` | `true` | Auto-reconnect to last used wallet |
| `enableBurnerWallet` | `false` | Enable dev-only burner wallet |
| `storage` | `localStorage` | Where to persist last connected wallet |

### Entry Point (`src/main.tsx`)

```tsx
import { DAppKitProvider } from '@mysten/dapp-kit-react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { dAppKit } from './dApp-kit';

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 2_000, refetchOnWindowFocus: true } },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <DAppKitProvider dAppKit={dAppKit}>
      <App />
    </DAppKitProvider>
  </QueryClientProvider>
);
```

### Game Constants (`src/constants.ts`)

```typescript
export const PACKAGE_ID = '0xYourPackageId';
export const WORLD_ID = '0xYourWorldObjectId';
export const GRID_ID = '0xYourGridObjectId';
export const GAME_SESSION_ID = '0xYourGameSessionId';
export const TURN_STATE_ID = '0xYourTurnStateId';
```

### Zustand UI Store (`src/stores/uiStore.ts`)

```typescript
import { create } from 'zustand';

interface UIStore {
  selectedEntity: string | null;
  selectEntity: (id: string | null) => void;
  isPending: boolean;
  setIsPending: (v: boolean) => void;
}

export const useUIStore = create<UIStore>((set) => ({
  selectedEntity: null,
  selectEntity: (id) => set({ selectedEntity: id }),
  isPending: false,
  setIsPending: (v) => set({ isPending: v }),
}));
```

### TypeScript Config

**tsconfig.json:**
```json
{ "files": [], "references": [{ "path": "./tsconfig.app.json" }, { "path": "./tsconfig.node.json" }] }
```

**tsconfig.app.json:**
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

**tsconfig.node.json:**
```json
{
  "compilerOptions": {
    "target": "ES2023", "lib": ["ES2023"],
    "module": "ESNext", "moduleResolution": "bundler",
    "strict": true, "noEmit": true, "skipLibCheck": true,
    "verbatimModuleSyntax": true, "types": ["node"]
  },
  "include": ["vite.config.ts"]
}
```

### Vite Config (`vite.config.ts`)

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': '/src' } },
  build: { target: 'esnext' }, // gRPC needs modern JS
});
```

### MVR Overrides (for `@mysten/codegen`)

```typescript
createClient(network) {
  return new SuiGrpcClient({
    network, baseUrl: GRPC_URLS[network],
    mvr: { overrides: { packages: { '@local-pkg/mycontract': PACKAGE_IDS[network] } } },
  });
}
```

---

## 2. Clients — Dual-Client Pattern

| Client | Import | Use For |
|--------|--------|---------|
| `SuiGrpcClient` | `@mysten/sui/grpc` | Wallet/tx via dApp Kit |
| `SuiJsonRpcClient` | `@mysten/sui/jsonRpc` | Data reads (`getObject`, `getDynamicFields`) |
| `SuiGraphQLClient` | `@mysten/sui/graphql` | Complex filtered/nested queries |

> ⚠️ `SuiClient` from `@mysten/sui/client` is **deprecated**. Use `SuiJsonRpcClient` from `@mysten/sui/jsonRpc`.

```typescript
// src/lib/suiClient.ts — standalone data reader
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

export const suiClient = new SuiJsonRpcClient({
  network: 'testnet',
  url: getJsonRpcFullnodeUrl('testnet'),
});
```

### Client API Quick Reference

```typescript
// gRPC (via useCurrentClient() or dApp Kit) — uses .core namespace
await client.core.getObject({ objectId, options: { showContent: true } });
await client.core.multiGetObjects({ ids, options: { showContent: true } });
await client.core.getOwnedObjects({ owner, filter, options });
await client.core.getBalance({ owner, coinType? });
await client.core.getAllBalances({ owner });
await client.core.getCoins({ owner, coinType, cursor, limit });
await client.core.queryEvents({ query, cursor, limit });
await client.waitForTransaction({ digest, include: { effects: true } });

// gRPC native services (advanced)
await client.stateService.listOwnedObjects({ owner: '0x...' });
await client.ledgerService.getTransaction({ digest: '...' });

// JSON-RPC (standalone — also has .core namespace, native methods use `id` not `objectId`)
await suiClient.getObject({ id, options: { showContent: true } });  // native
await suiClient.core.getObject({ objectId, options: { showContent: true } });  // core API
await suiClient.getDynamicFields({ parentId });  // ← NOT available on gRPC
await suiClient.multiGetObjects({ ids, options: { showContent: true } });
await suiClient.getOwnedObjects({ owner });

// GraphQL (complex queries)
import { SuiGraphQLClient } from '@mysten/sui/graphql';
import { graphql } from '@mysten/sui/graphql/schemas/latest';
const gqlClient = new SuiGraphQLClient({ url: 'https://sui-testnet.mystenlabs.com/graphql' });
const result = await gqlClient.execute(graphql(`query { ... }`), vars);
```

### Network Endpoints

| Network | gRPC Base URL |
|---------|--------------|
| Mainnet | `https://fullnode.mainnet.sui.io:443` |
| Testnet | `https://fullnode.testnet.sui.io:443` |
| Devnet | `https://fullnode.devnet.sui.io:443` |
| Localnet | `http://127.0.0.1:9000` |

> ⚠️ Public endpoints are rate-limited. Use Shinami, BlockVision, etc. in production.

---

## 3. React Hooks (`@mysten/dapp-kit-react`)

| Hook | Returns | Use For |
|------|---------|---------|
| `useDAppKit()` | Full DAppKit instance | `signAndExecuteTransaction`, `connectWallet`, `switchNetwork` |
| `useCurrentAccount()` | `UiWalletAccount \| null` | Connected address, public key |
| `useCurrentWallet()` | `UiWallet \| null` | Wallet name, icon, accounts |
| `useCurrentClient()` | Configured client | Network-aware data reads |
| `useCurrentNetwork()` | `'testnet' \| 'mainnet'...` | Active network string |
| `useWalletConnection()` | `{ status, isConnected, account }` | Connection state + boolean flags |
| `useWallets()` | `UiWallet[]` | All detected compatible wallets |

### `useWalletConnection()` States

| Status | `wallet` | `account` | Boolean Flags |
|--------|----------|-----------|---------------|
| `'disconnected'` | `null` | `null` | `isDisconnected: true` |
| `'connecting'` | `null` | `null` | `isConnecting: true` |
| `'reconnecting'` | `UiWallet` | `UiWalletAccount` | `isReconnecting: true` |
| `'connected'` | `UiWallet` | `UiWalletAccount` | `isConnected: true` |

### Wallet Components & Patterns

```tsx
import { ConnectButton, ConnectModal } from '@mysten/dapp-kit-react/ui';

// All-in-one: connect/disconnect/account-switch
<ConnectButton />

// Standalone modal
<ConnectModal />
```

### Manual Wallet Connection

```typescript
const dAppKit = useDAppKit();
const wallets = useWallets();

// Connect
const { accounts } = await dAppKit.connectWallet({ wallet: wallets[0] });

// Disconnect
await dAppKit.disconnectWallet();

// Switch account / network
dAppKit.switchAccount({ account: targetAccount });
dAppKit.switchNetwork('mainnet');

// Sign personal message (for auth / off-chain verification)
const { bytes, signature } = await dAppKit.signPersonalMessage({
  message: new TextEncoder().encode('Sign this'),
});

// Get client for specific network
const testnetClient = dAppKit.getClient('testnet');

// Nanostore subscription (advanced, non-React contexts)
const unsub = dAppKit.stores.$connection.subscribe((conn) => { /* ... */ });
```

### Wallet-Not-Installed Handling

```tsx
const wallets = useWallets();
if (wallets.length === 0) {
  return <a href="https://slush.app" target="_blank">Install Slush Wallet</a>;
}
```

---

## 4. Transactions (PTBs)

### Build & Execute Pattern

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { useQueryClient } from '@tanstack/react-query';

export function useGameActions() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();
  const queryClient = useQueryClient();

  async function executeAndRefresh(tx: Transaction) {
    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw parseTransactionError(result.FailedTransaction);
    }

    await client.waitForTransaction({
      digest: result.Transaction.digest,
      include: { effects: true },
    });

    // refetchQueries forces immediate re-fetch (use over invalidateQueries for games)
    await Promise.all([
      queryClient.refetchQueries({ queryKey: ['gameSession'] }),
      queryClient.refetchQueries({ queryKey: ['grid'] }),
      queryClient.refetchQueries({ queryKey: ['turnState'] }),
    ]);
    return result.Transaction;
  }

  return {
    moveEntity: (entityId: string, x: number, y: number) => {
      const tx = new Transaction();
      tx.moveCall({
        target: `${PACKAGE_ID}::game::move_entity`,
        arguments: [tx.object(WORLD_ID), tx.object(GRID_ID), tx.object(entityId),
                    tx.pure.u64(x), tx.pure.u64(y)],
      });
      return executeAndRefresh(tx);
    },
    // ... other actions follow same pattern
  };
}
```

### PTB Commands

```typescript
// Move call (target = 'packageId::module::function')
tx.moveCall({ target: '0xPkg::module::function', typeArguments?, arguments });

// Coin operations
const [coin] = tx.splitCoins(tx.gas, [100_000_000]); // split from gas
tx.mergeCoins(targetCoin, [coin1, coin2]);             // merge coins
tx.transferObjects([coin], '0xRecipient');              // transfer

// coinWithBalance helper (simpler than splitCoins for transfers)
import { coinWithBalance } from '@mysten/sui/transactions';
tx.transferObjects([coinWithBalance({ balance: 100_000_000 })], '0xRecipient');

// Vectors
const vec = tx.makeMoveVec({ type: '0x2::coin::Coin<0x2::sui::SUI>', elements: [coin1, coin2] });

// Gas / sender
tx.setGasBudget(10_000_000);
tx.setSender('0xSponsorAddress'); // sponsored txs

// Sign only (without executing)
const signedTx = await dAppKit.signTransaction({ transaction: tx });
```

### `tx.pure.*` Typed Helpers

```typescript
tx.pure.u8(255)       tx.pure.u64(1000n)     tx.pure.bool(true)
tx.pure.string('hi')  tx.pure.address('0x…') tx.pure.id('0x…')
tx.pure.option('u64', 42)                     tx.pure.vector('u8', [1,2,3])

// String type arg syntax (equivalent to methods above)
tx.pure('vector<u8>', [1, 2, 3])
tx.pure('option<u8>', 1)
tx.pure('vector<option<u8>>', [1, null, 2])
```

### Object References

```typescript
tx.object('0xId')                // auto-resolves mutability & receiving
tx.object.shared('0xId')         // shared object (explicit)
tx.object.receiving('0xId')      // receiving object (explicit)
tx.object.system()               // 0x5 (SuiSystemState)
tx.object.clock()                // 0x6 (sui::clock::Clock)
tx.object.random()               // 0x8 (0x2::random::Random)
tx.object.denyList()             // DenyList object
tx.object.option({ type: '0xPkg::module::T', value: '0xId' })  // Option<T> object
tx.splitCoins(tx.gas, [100_000_000])  // split from gas

// Pre-resolved refs (skip RPC lookups for performance)
import { Inputs } from '@mysten/sui/transactions';
tx.object(Inputs.ObjectRef({ digest, objectId, version }));        // owned/immutable
tx.object(Inputs.SharedObjectRef({ objectId, initialSharedVersion, mutable })); // shared
tx.object(Inputs.ReceivingRef({ digest, objectId, version }));     // receiving
```

### Multi-Call PTB (batch actions)

```typescript
const tx = new Transaction();
tx.moveCall({ target: `${PACKAGE_ID}::game::move_entity`, arguments: [...] });
tx.moveCall({ target: `${PACKAGE_ID}::game::attack`, arguments: [...] });
tx.moveCall({ target: `${PACKAGE_ID}::game::end_turn`, arguments: [...] });
return executeAndRefresh(tx); // all 3 calls in ONE transaction
```

### Codegen Pattern (`tx.add()`)

```typescript
import { increment } from './contracts/counter/counter';
const tx = new Transaction();
tx.add(increment({ arguments: [counterId] }));
```

### Extracting Created Objects from Effects

```typescript
const txResult = await client.waitForTransaction({
  digest: result.Transaction.digest,
  include: { effects: true },
});
if (txResult.Transaction) {
  const created = txResult.Transaction.effects?.changedObjects?.find(
    (obj: any) => obj.idOperation === 'Created',
  );
  console.log('Created ID:', created?.objectId);
}
```

### Error Handling — Map Abort Codes

```typescript
const GAME_ERROR_MAP: Record<number, string> = {
  101: "It's not your turn",
  102: 'Game is full',
  // ... mirror Move contract error constants
};

export function parseTransactionError(failure: { error: string }): Error {
  const match = failure.error.match(/MoveAbort.*?(\d+)\)?$/);
  if (match) {
    const code = Number(match[1]);
    return new Error(GAME_ERROR_MAP[code] ?? `Transaction failed (code ${code})`);
  }
  return new Error(failure.error || 'Transaction failed');
}
```

---

## 5. Reading On-Chain Data

### Game Session (shared object polling)

```typescript
export function useGameState() {
  return useQuery<GameSession>({
    queryKey: ['gameSession', GAME_SESSION_ID],
    queryFn: async () => {
      const res = await suiClient.getObject({
        id: GAME_SESSION_ID,
        options: { showContent: true },
      });
      if (res.data?.content?.dataType !== 'moveObject') throw new Error('Not found');
      return parseGameSession(res.data.content.fields as Record<string, any>);
    },
    refetchInterval: 3_000,
  });
}
```

### Grid (shared object)

```typescript
export function useGrid() {
  return useQuery({
    queryKey: ['grid', GRID_ID],
    queryFn: async () => {
      const res = await suiClient.getObject({ id: GRID_ID, options: { showContent: true } });
      const fields = res.data?.content?.fields as Record<string, any>;
      return { width: Number(fields.width), height: Number(fields.height), cells: parseCells(fields.cells) };
    },
    refetchInterval: 3_000,
  });
}
```

### Turn State + isMyTurn

```typescript
export function useTurnState() {
  const account = useCurrentAccount();
  const query = useQuery({
    queryKey: ['turnState', TURN_STATE_ID],
    queryFn: async () => {
      const res = await suiClient.getObject({ id: TURN_STATE_ID, options: { showContent: true } });
      const fields = res.data?.content?.fields as Record<string, any>;
      return { currentPlayer: Number(fields.current_player), turnNumber: Number(fields.turn_number) };
    },
    refetchInterval: 2_000,
  });
  const isMyTurn = query.data ? account !== null /* derive from player index */ : false;
  return { ...query, isMyTurn };
}
```

### ECS Entity Components (dynamic fields)

> **Must use `SuiJsonRpcClient`** — gRPC does not expose `getDynamicFields`.

```typescript
export function usePlayerEntity(entityId: string | null) {
  return useQuery({
    queryKey: ['playerEntity', entityId],
    queryFn: async () => {
      // 1. Discover components
      const dynFields = await suiClient.getDynamicFields({ parentId: entityId! });
      // 2. Batch-fetch
      const objects = await suiClient.multiGetObjects({
        ids: dynFields.data.map(df => df.objectId),
        options: { showContent: true, showType: true },
      });
      // 3. Parse by type name
      for (const obj of objects) {
        const typeName = obj.data?.content?.type ?? '';
        const fields = obj.data?.content?.fields as Record<string, any>;
        const value = fields.value?.fields ?? fields.value ?? fields;
        if (typeName.includes('Health')) health = { current: Number(value.current), max: Number(value.max) };
        // ... pattern-match other component types
      }
      return { health, energy, gold, deck };
    },
    enabled: !!entityId,
    refetchInterval: 2_000,
  });
}
```

### Discovering Player Entities (owned objects)

```typescript
export function useMyEntities(packageId: string) {
  const account = useCurrentAccount();
  return useQuery({
    queryKey: ['myEntities', account?.address],
    queryFn: async () => {
      const res = await suiClient.getOwnedObjects({
        owner: account!.address,
        filter: { StructType: `${packageId}::entity::Entity` },
        options: { showContent: true },
      });
      return res.data?.map(obj => parsePlayerEntity(obj.data?.content?.fields as Record<string, any>)) ?? [];
    },
    enabled: !!account,
    refetchInterval: 5_000,
  });
}
```

### Batch Multiple Objects

```typescript
const [session, grid, turnState] = await Promise.all([
  suiClient.getObject({ id: GAME_SESSION_ID, options: { showContent: true } }),
  suiClient.getObject({ id: GRID_ID, options: { showContent: true } }),
  suiClient.getObject({ id: TURN_STATE_ID, options: { showContent: true } }),
]);
```

### BCS Deserialization (nested structs, vectors, Options)

When parsed JSON is unreliable (nested structs, `vector<Struct>`, `Option<T>`), use BCS:

```typescript
import { bcs, fromBase64, type InferBcsType } from '@mysten/bcs';

// UID helper
const UID = bcs.fixedArray(32, bcs.u8()).transform({
  input: (id: string) => Array.from(Buffer.from(id.replace('0x', ''), 'hex')),
  output: (bytes) => '0x' + Buffer.from(bytes).toString('hex'),
});

// Component schemas (field order MUST match Move struct)
const PositionSchema = bcs.struct('Position', { x: bcs.u64(), y: bcs.u64() });
const HealthSchema = bcs.struct('Health', { current: bcs.u64(), max: bcs.u64() });
const EnergySchema = bcs.struct('Energy', { current: bcs.u64(), max: bcs.u64(), regen: bcs.u64() });
const AttackSchema = bcs.struct('Attack', { value: bcs.u64(), range: bcs.u64() });
const DefenseSchema = bcs.struct('Defense', { armor: bcs.u64(), resistance: bcs.u64() });

// Nested: vector<Struct>
const EffectSchema = bcs.struct('Effect', { effect_type: bcs.u8(), stacks: bcs.u64(), duration: bcs.u8() });
const StatusEffectSchema = bcs.struct('StatusEffect', { effects: bcs.vector(EffectSchema) });

const CardDataSchema = bcs.struct('CardData', { card_id: bcs.u64(), cost: bcs.u64(), attack: bcs.u64(), defense: bcs.u64() });
const DeckSchema = bcs.struct('Deck', { draw_pile: bcs.vector(CardDataSchema), hand: bcs.vector(CardDataSchema), discard_pile: bcs.vector(CardDataSchema) });

// Option<address>
const GameSessionSchema = bcs.struct('GameSession', {
  id: UID, state: bcs.u8(),
  players: bcs.vector(bcs.fixedArray(32, bcs.u8())),
  max_players: bcs.u64(),
  winner: bcs.option(bcs.fixedArray(32, bcs.u8())),
});

type Health = InferBcsType<typeof HealthSchema>;

// Fetch with showBcs: true
const res = await client.core.getObject({ objectId, options: { showBcs: true } });
if (res.data?.bcs?.dataType === 'moveObject') {
  const parsed = HealthSchema.parse(fromBase64(res.data.bcs.bcsBytes));
}
```

| Use Case | Approach |
|----------|----------|
| Flat structs (u64, string) | `showContent: true` + `fields as Record<string, any>` |
| Nested structs, `vector<Struct>`, `Option<T>` | `showBcs: true` + BCS schema |
| Dynamic fields (ECS components) | `getDynamicFields` → `multiGetObjects` |

---

## 6. Real-Time & Polling Patterns

| Game Pace | Strategy | `refetchInterval` |
|-----------|----------|-------------------|
| Turn-based (slow) | Polling | 3–5s |
| Turn-based (fast) | Polling + event hint | 1–2s |
| Real-time | Event subscription + fallback | 1s |

### Adaptive Polling

```typescript
const pollInterval = { lobby: 5_000, active: 2_000, finished: 0 }[gamePhase];
useQuery({
  queryKey: ['game'], queryFn: fetchGame,
  refetchInterval: pollInterval || false,
  refetchIntervalInBackground: false, // stop polling when tab is hidden
});
```

### Optimistic Updates

```typescript
// 1. Immediately update cache
queryClient.setQueryData(['grid', GRID_ID], (old) => ({ ...old, /* optimistic change */ }));
// 2. Execute real tx
try { await moveEntity(entityId, x, y); }
// 3. Rollback on failure
catch { queryClient.invalidateQueries({ queryKey: ['grid'] }); throw err; }
```

### Events — For Activity Logs, Not State

```typescript
// Event types (discriminated union matching Move event structs)
type GameEvent =
  | { type: 'GameCreated'; data: { game_id: string; creator: string }; timestamp: string }
  | { type: 'PlayerJoined'; data: { game_id: string; player: string }; timestamp: string }
  | { type: 'GameStarted'; data: { game_id: string }; timestamp: string }
  | { type: 'GameOver'; data: { game_id: string; winner: string }; timestamp: string };

function parseEventType(typeRepr: string): string | null {
  const match = typeRepr.match(/::(\w+)$/);
  return match ? match[1] : null;
}

export function useGameLog() {
  const client = useCurrentClient();
  return useQuery<GameEvent[]>({
    queryKey: ['gameLog'],
    queryFn: async () => {
      const res = await client.core.queryEvents({
        query: { MoveModule: { package: PACKAGE_ID, module: 'game' } },
        order: 'descending', limit: 50,
      });
      return res.data.map(e => {
        const name = parseEventType(e.type);
        return name ? { type: name, data: e.parsedJson, timestamp: e.timestampMs ?? '' } : null;
      }).filter(Boolean) as GameEvent[];
    },
    refetchInterval: 5_000,
  });
}
```

### Event-Driven Notifications

```typescript
export function useGameNotifications() {
  const { data: events } = useGameLog();
  const account = useCurrentAccount();
  const lastSeenCount = useRef(0);

  useEffect(() => {
    if (!events || events.length <= lastSeenCount.current) return;
    const newEvents = events.slice(0, events.length - lastSeenCount.current);
    lastSeenCount.current = events.length;
    for (const event of newEvents) {
      if (event.type === 'GameOver' && event.data.winner === account?.address) showToast('🎉 You won!');
    }
  }, [events, account]);
}
```

### Turn Change Detection

```typescript
function useTurnNotification() {
  const { isMyTurn } = useTurnState();
  const prevIsMyTurn = useRef(false);
  useEffect(() => {
    if (isMyTurn && !prevIsMyTurn.current) { playSound('your-turn.mp3'); }
    prevIsMyTurn.current = isMyTurn;
  }, [isMyTurn]);
}
```

### Stale Data Warning

```tsx
const { data, dataUpdatedAt, isFetching } = useGameState();
const isStale = Date.now() - dataUpdatedAt > 10_000;
{isStale && <div className="stale-warning">Data may be outdated</div>}
```

---

## 7. Contract → Frontend Integration

### Move → TypeScript Type Map

| Move | TypeScript | Parser |
|------|-----------|--------|
| `u8/u16/u32` | `number` | `Number(field)` |
| `u64/u128/u256` | `number` | `Number(field)` |
| `bool` | `boolean` | `Boolean(field)` |
| `address`, `ID` | `string` | direct |
| `vector<u8>` | `number[]` | `.map(Number)` |
| `vector<address>` | `string[]` | direct |
| `Option<T>` | `T \| null` | `field?.Some ?? null` |
| `String` | `string` | direct |

### Move Function → PTB Argument Map

| Move Param | PTB Argument |
|-----------|-------------|
| `&World`, `&mut Object` | `tx.object(OBJECT_ID)` |
| `u8` | `tx.pure.u8(val)` |
| `u64` | `tx.pure.u64(val)` |
| `address` | `tx.pure.address(val)` |
| `&Clock` | `tx.object('0x6')` |
| `&TxContext` | **skip** (auto-injected) |

### Naming Convention

- Move `snake_case` fields → TypeScript `camelCase` properties
- `UID` → `id: string` (use objectId from query)
- `u8` state fields → TypeScript string union (map via constants)

### Parser Pattern

```typescript
export function parseGameSession(objectId: string, fields: Record<string, any>): GameSession {
  return {
    id: objectId,
    state: parseState(Number(fields.state)),
    player: fields.player,
    board: (fields.board as any[]).map(Number),        // vector<u8>
    movesRemaining: Number(fields.moves_remaining),    // snake → camelCase
    winner: fields.winner?.Some ?? null,               // Option<T>
  };
}
```

---

## 8. UI Patterns

### App Shell — Game Phase Routing

```tsx
function App() {
  const account = useCurrentAccount();
  const { data: game, isLoading } = useGameState();

  if (!account) return <WelcomeScreen />;
  if (isLoading) return <LoadingScreen />;
  switch (game?.state) {
    case 'lobby': return <GameLobby />;
    case 'active': return <GameBoard />;
    case 'finished': return <GameOver />;
  }
}
```

### Grid Board (CSS Grid)

```tsx
<div className="game-grid" style={{
  display: 'grid',
  gridTemplateColumns: `repeat(${width}, 1fr)`,
  gap: '2px', maxWidth: '600px',
}}>
  {cells.map(cell => <GridCell key={`${cell.x}-${cell.y}`} cell={cell} />)}
</div>
```

```css
.grid-cell {
  aspect-ratio: 1;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 4px;
  display: flex; align-items: center; justify-content: center;
  cursor: pointer; transition: background-color 0.15s ease;
}
.grid-cell:hover { background-color: rgba(255, 255, 255, 0.1); }
.grid-cell--occupied { background-color: rgba(59, 130, 246, 0.3); }
.grid-cell--selected { outline: 2px solid #3b82f6; outline-offset: -2px; }
.grid-cell--reachable { background-color: rgba(34, 197, 94, 0.2); }
```

### Player HUD with Progress Bar

```tsx
function PlayerHud({ entity }: { entity: PlayerEntity }) {
  return (
    <div className="player-hud">
      <ProgressBar label="HP" current={entity.health.current} max={entity.health.max} color="red" />
      <ProgressBar label="Energy" current={entity.energy.current} max={entity.energy.max} color="blue" />
    </div>
  );
}

function ProgressBar({ label, current, max, color }) {
  const pct = Math.round((current / max) * 100);
  return (
    <div className="stat-row">
      <label>{label}</label>
      <div className="progress-bar">
        <div className="progress-bar__fill" style={{ width: `${pct}%`, backgroundColor: `var(--color-${color})` }} />
        <span>{current}/{max}</span>
      </div>
    </div>
  );
}
```

### Action Button with Loading

```tsx
function ActionButton({ label, onClick, disabled }) {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState(null);
  async function handleClick() {
    setIsPending(true); setError(null);
    try { await onClick(); }
    catch (err) { setError(err.message); }
    finally { setIsPending(false); }
  }
  return <button onClick={handleClick} disabled={disabled || isPending}>
    {isPending ? `${label}...` : label}
  </button>;
}
```

### GameLoader (preload before interaction)

```tsx
function GameLoader({ children }: { children: React.ReactNode }) {
  const gameState = useGameState();
  const grid = useGrid();
  if (!gameState.data || !grid.data) return <LoadingScreen />;
  return <>{children}</>;
}
```

### Performance Tips

1. **`React.memo`** grid cells — they re-render on every poll
2. **Split queries by frequency** — turn state (2s), grid (3s), session (5s)
3. **Don't use `useDAppKit()` in leaf components** — lift to parent, pass callbacks
4. **CSS transitions** over JS animation for entity movement
5. **Preload** game objects on mount before user interacts

---

## 9. Utilities (`@mysten/sui/utils`)

```typescript
import { formatAddress, formatDigest, MIST_PER_SUI, SUI_DECIMALS,
  isValidSuiAddress, isValidSuiObjectId, isValidTransactionDigest,
  normalizeSuiAddress, normalizeStructTag, parseStructTag,
  fromHex, toHex, fromBase64, toBase64 } from '@mysten/sui/utils';

formatAddress('0x1234...abcdef')   // '0x1234...abcd'
normalizeSuiAddress('0x2')         // '0x0000...0002' (64-char hex)
MIST_PER_SUI                      // 1_000_000_000n
SUI_DECIMALS                      // 9

// SUI ↔ MIST conversion
const mist = BigInt(Math.floor(sui * Number(MIST_PER_SUI)));
const sui = (Number(mistBigInt) / Number(MIST_PER_SUI)).toFixed(9);

// Struct tag parsing
const parsed = parseStructTag('0x2::coin::Coin<0x2::sui::SUI>');
// { address: '0x2', module: 'coin', name: 'Coin', typeParams: [...] }
```

| Constant | Value |
|----------|-------|
| `SUI_CLOCK_OBJECT_ID` | `'0x6'` |
| `SUI_SYSTEM_STATE_OBJECT_ID` | `'0x5'` |
| `SUI_RANDOM_OBJECT_ID` | `'0x8'` (`0x2::random::Random`) |
| `SUI_TYPE_ARG` | `'0x2::sui::SUI'` |

---

## 10. Common Recipes

### Send SUI

```typescript
const tx = new Transaction();
const [coin] = tx.splitCoins(tx.gas, [0.5 * Number(MIST_PER_SUI)]);
tx.transferObjects([coin], '0xRecipient');
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
```

### Check Balance

```typescript
const balance = await client.core.getBalance({ owner: account.address });
const sui = Number(BigInt(balance.totalBalance)) / Number(MIST_PER_SUI);
```

### Transfer an Owned Object

```typescript
const tx = new Transaction();
tx.transferObjects([tx.object(objectId)], '0xRecipient');
```

### List Owned Objects (filtered by type)

```typescript
const res = await client.core.getOwnedObjects({
  owner: account.address,
  filter: { StructType: '0xPkg::module::MyStruct' },
  options: { showContent: true, showType: true },
});
```

---

## 11. Do's and Don'ts

### ❌ Don'ts

| Don't | Do Instead |
|-------|-----------|
| `import { ... } from '@mysten/dapp-kit'` | `from '@mysten/dapp-kit-react'` |
| `import { SuiClient } from '@mysten/sui/client'` | `SuiGrpcClient` or `SuiJsonRpcClient` |
| Assume tx success: `result.Transaction.digest` | Check `if (result.FailedTransaction)` first |
| Read state immediately after tx | `await client.waitForTransaction(...)` first |
| Create clients inside components | Use `useCurrentClient()` hook |
| Skip type registration | Add `declare module '@mysten/dapp-kit-react'` block |
| Use BCS when `tx.pure.*` exists | Use `tx.pure.u64(42)` not `bcs.u64().serialize(42n)` |
| Use public endpoints in production | Use Shinami, BlockVision, etc. |
| Use `useDAppKit()` in leaf components | Use `useCurrentClient()` or lift to parent |

### ✅ Do's

- Always check `result.FailedTransaction` after `signAndExecuteTransaction`
- Always `waitForTransaction` before reading post-tx state
- Use `formatAddress()` for display
- Use `enableBurnerWallet: import.meta.env.DEV` (dev only)
- Use `refetchQueries` (not `invalidateQueries`) for games
- Handle "no wallets detected" with install prompt
- Use `tx.add()` with codegen-generated functions

---

## 12. Scaffolding Protocol (File Creation Order)

### Phase 1: Contract Analysis
Read `sources/game.move` → extract structs, consts, public fns, shared objects.

### Phase 2: Scaffold Boilerplate (templates, unchanged)

| # | File | Notes |
|---|------|-------|
| 1 | `package.json` | Replace `__GAME_SLUG__` |
| 2 | `vite.config.ts` | `target: 'esnext'` |
| 3 | `index.html` | Replace `__GAME_TITLE__` |
| 4 | `tsconfig.json` + `tsconfig.app.json` + `tsconfig.node.json` | |
| 5 | `src/main.tsx` | Providers (unchanged) |
| 6 | `src/dApp-kit.ts` | createDAppKit + type registration |
| 7 | `src/lib/suiClient.ts` | SuiJsonRpcClient |
| 8 | `src/stores/uiStore.ts` | Zustand UI state |

### Phase 3: Contract Integration (generated from contract)

| # | File | Source |
|---|------|--------|
| 9 | `src/constants.ts` | Contract consts + deployed IDs |
| 10 | `src/lib/types.ts` | Contract structs → TS interfaces |
| 11 | `src/lib/parsers.ts` | Field mapping functions |

### Phase 4: Hooks

| # | File |
|---|------|
| 12 | `src/hooks/useGameActions.ts` |
| 13 | `src/hooks/useGameSession.ts` |

### Phase 5: Game UI

| # | File |
|---|------|
| 14 | `src/App.tsx` |
| 15 | `src/index.css` |
| 16+ | `src/components/*.tsx` |

### Phase 6: Verify
`npm install && npm run dev` → check console, wallet connect, state reads, game actions.

---

## 13. Game Catalog & Pattern Index

### Existing Games

| Game | Type | Grid? | Components | Key Patterns |
|------|------|-------|------------|--------------|
| virus_game | Puzzle | ✅ 5×5→12×12 | 7 | Multi-step PTB, Phaser renderer, local engine |
| sokoban | Puzzle | ✅ 6×6 | 5 | Local simulation, level data, directional input |
| card_crawler | Roguelike | ✅ 3×3 | 13 | Card system, animations, entity components |
| tactics_ogre | Tactics | ✅ variable | 9 | Multi-entity, turn-based, 6 hooks |
| tetris_game | Arcade | ✅ 10×20 | 3 | Real-time tick, keyboard input |
| maze_game | Puzzle | ✅ variable | 3 | Pathfinding, fog of war |
| flappy_bird | Arcade | ❌ | 3 | Physics loop, collision detection |
| sandbox_game | Sandbox | ✅ variable | 3 | Entity editor, creative mode |

### Game Complexity Reference

| Complexity | Examples | Components | Hooks |
|-----------|---------|------------|-------|
| Simple | flappy_bird, maze | 3 | 1 |
| Medium | sokoban, virus_game | 5-7 | 2 |
| Complex | card_crawler, tactics_ogre | 9-13 | 4-6 |

### Pattern Quick Reference

| Pattern | Best Example | Key Idea |
|---------|-------------|----------|
| Grid board (CSS) | sokoban, card_crawler | `grid-template-columns: repeat(N,1fr)` |
| Grid board (Canvas) | virus_game | Phaser for animations/particles |
| Multi-step PTB | virus_game | Chain `moveCall`s in one tx |
| Local simulation | virus_game, sokoban | Preview moves client-side before on-chain |
| Level selection | virus_game, sokoban | Level metadata in `constants.ts` |
| Card-based UI | card_crawler | `cardLookup.ts` + card components |
| Real-time/physics | flappy_bird, tetris | `requestAnimationFrame` / `setInterval` |
| Animation-heavy | card_crawler | Separate `animations.css` + keyframes |
| Multi-entity turns | tactics_ogre | 6 hooks, turn management, action selection |
