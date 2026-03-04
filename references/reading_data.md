# Reading Data from Sui

## Getting the Client

For data reads, use `useCurrentClient()` from dApp Kit or a standalone `SuiGrpcClient`:

```typescript
import { useCurrentClient } from '@mysten/dapp-kit-react';
const client = useCurrentClient(); // gRPC client from dApp Kit
```

For wallet-aware operations, use the dApp Kit client:

```typescript
import { useCurrentClient } from '@mysten/dapp-kit-react';

function DataReader() {
  const client = useCurrentClient();
  // client is a SuiGrpcClient (or whatever you configured)
}
```

## Core API (Transport-Agnostic)

The `client.core` namespace provides a consistent API across gRPC and GraphQL clients.

### Get Object

```typescript
const object = await client.core.getObject({
  objectId: '0x...',
  options: { showContent: true },
});

if (object.data) {
  console.log('Type:', object.data.type);
  console.log('Content:', object.data.content);
}
```

### Get Multiple Objects

```typescript
const objects = await client.core.multiGetObjects({
  ids: ['0xObj1', '0xObj2'],
  options: { showContent: true },
});
```

### Get Balance

```typescript
const balance = await client.core.getBalance({
  owner: '0xOwnerAddress',
  // coinType: '0x2::sui::SUI', // optional, defaults to SUI
});

console.log('Total:', balance.totalBalance); // string of MIST
```

### Get All Balances

```typescript
const balances = await client.core.getAllBalances({
  owner: '0xOwnerAddress',
});
// Returns array of { coinType, totalBalance, coinObjectCount, ... }
```

### Get Coins

```typescript
const coins = await client.core.getCoins({
  owner: '0xOwnerAddress',
  coinType: '0x2::sui::SUI',
});
// coins.data → array of coin objects
// coins.nextCursor → for pagination
```

### Get Owned Objects

```typescript
const ownedObjects = await client.core.getOwnedObjects({
  owner: '0xOwnerAddress',
  filter: { StructType: '0xPkg::module::MyStruct' },
  options: { showContent: true },
});
```

### Wait for Transaction

```typescript
const txResult = await client.core.waitForTransaction({
  digest: '...',
  include: { effects: true },
});
```

## gRPC Native API

`SuiGrpcClient` provides direct access to gRPC service clients for advanced queries:

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';

const client = new SuiGrpcClient({ network: 'testnet', baseUrl: 'https://fullnode.testnet.sui.io:443' });

// State service (uses { response } destructuring)
const { response } = await client.stateService.listOwnedObjects({ owner: '0x...' });
const { response: fields } = await client.stateService.listDynamicFields({ parent: '0x...' });

// Ledger service
const { response: tx } = await client.ledgerService.getTransaction({ digest: '...' });
```

## GraphQL Client (Advanced)

For complex queries, use `SuiGraphQLClient`:

```typescript
import { SuiGraphQLClient } from '@mysten/sui/graphql';
import { graphql } from '@mysten/sui/graphql/schemas/latest';

const gqlClient = new SuiGraphQLClient({ url: 'https://sui-testnet.mystenlabs.com/graphql' });

const query = graphql(`
  query GetObject($id: SuiAddress!) {
    object(address: $id) {
      objectId
      version
      digest
      owner { ... on AddressOwner { owner { address } } }
    }
  }
`);

const result = await gqlClient.execute(query, { id: '0x...' });
```

## Parsing Move Struct Fields

When using `showContent: true`, object content is returned as parsed JSON:

```typescript
const obj = await client.core.getObject({
  objectId: '0x...',
  options: { showContent: true },
});

if (obj.data?.content?.dataType === 'moveObject') {
  const fields = obj.data.content.fields as Record<string, any>;
  console.log('Counter value:', fields.value);
  console.log('Owner:', fields.owner);
}
```

## Using with React

### Pattern: Fetch on Mount

```typescript
function ObjectViewer({ objectId }: { objectId: string }) {
  const client = useCurrentClient();
  const [data, setData] = useState<any>(null);

  useEffect(() => {
    client.core.getObject({ objectId, options: { showContent: true } })
      .then((result) => setData(result.data))
      .catch(console.error);
  }, [client, objectId]);

  if (!data) return <p>Loading...</p>;
  return <pre>{JSON.stringify(data.content?.fields, null, 2)}</pre>;
}
```

### Pattern: With React Query

```typescript
import { useQuery } from '@tanstack/react-query';

function BalanceDisplay({ address }: { address: string }) {
  const client = useCurrentClient();

  const { data: balance } = useQuery({
    queryKey: ['balance', address],
    queryFn: () => client.core.getBalance({ owner: address }),
  });

  return <p>Balance: {balance?.totalBalance ?? '...'} MIST</p>;
}
```

---

## Game State Reading Patterns

### Reading Game Session

```typescript
// src/hooks/useGameState.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { GAME_SESSION_ID } from '../constants';
import { parseGameSession, type GameSession } from '../lib/parsers';

export function useGameState() {
  const client = useCurrentClient();
  return useQuery<GameSession>({
    queryKey: ['gameSession', GAME_SESSION_ID],
    queryFn: async () => {
      const { object } = await client.core.getObject({
        objectId: GAME_SESSION_ID,
        include: { json: true },
      });
      if (!object?.json) {
        throw new Error('GameSession not found');
      }
      return parseGameSession(object.json as Record<string, any>);
    },
    refetchInterval: 3_000, // Poll every 3 seconds during active gameplay
  });
}
```

### Reading an Entity (ECS Components via Dynamic Fields)

ECS entities store components as dynamic fields in a `Bag`. A simple `getObject` won't return them — you need `listDynamicFields` → `getObjects` → type-match.

> [!IMPORTANT]
> Use `client.core.listDynamicFields()` for entity reads. This is available on `SuiGrpcClient` Core API.

```typescript
// src/hooks/usePlayerEntity.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { parseHealthComponent, parseEnergyComponent, parseDeckComponent } from '../lib/parsers';
import type { PlayerState } from '../lib/types';

export function usePlayerEntity(entityId: string | null) {
  const client = useCurrentClient();
  return useQuery<PlayerState>({
    queryKey: ['playerEntity', entityId],
    queryFn: async () => {
      // Step 1: Discover all dynamic fields (components) on the entity
      const dynFields = await client.core.listDynamicFields({ parentId: entityId! });
      if (!dynFields.dynamicFields?.length) {
        return defaultPlayerState(entityId!);
      }

      // Step 2: Fetch each component via getDynamicField
      // listDynamicFields returns { name, type } per entry — NOT objectId.
      // Use getDynamicField to get the actual value for each component.
      let health = { current: 0, max: 0 };
      let energy = { current: 0, max: 0, regen: 0 };
      let gold = 0;
      let deck = { drawPile: [], hand: [], discardPile: [] };

      for (const field of dynFields.dynamicFields) {
        try {
          const { dynamicField } = await client.core.getDynamicField({
            parentId: entityId!,
            name: field.name,
          });
          const typeName = dynamicField.value?.type ?? '';
          // BCS decode or parse JSON depending on your needs
          if (typeName.includes('Health')) {
            health = parseHealthComponent(dynamicField.value);
          } else if (typeName.includes('Energy')) {
            energy = parseEnergyComponent(dynamicField.value);
          } else if (typeName.includes('Gold')) {
            gold = Number(dynamicField.value);
          } else if (typeName.includes('Deck')) {
            deck = parseDeckComponent(dynamicField.value);
          }
        } catch { /* field may not exist */ }
      }

      return { id: entityId!, health, energy, gold, deck };
    },
    enabled: !!entityId,
    refetchInterval: 2_000,
  });
}
```

### Reading the Grid

The grid is typically a shared object with dynamic fields for cells:

```typescript
// src/hooks/useGrid.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { GRID_ID } from '../constants';

export function useGrid() {
  const client = useCurrentClient();
  return useQuery({
    queryKey: ['grid', GRID_ID],
    queryFn: async () => {
      const { object } = await client.core.getObject({
        objectId: GRID_ID,
        include: { json: true },
      });
      if (!object?.json) {
        throw new Error('Grid not found');
      }
      const fields = object.json as Record<string, any>;
      return {
        width: Number(fields.width),
        height: Number(fields.height),
        cells: parseCells(fields.cells), // parse nested cell data
      };
    },
    refetchInterval: 3_000,
  });
}
```

### Reading Turn State

```typescript
// src/hooks/useTurnState.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentAccount, useCurrentClient } from '@mysten/dapp-kit-react';
import { TURN_STATE_ID } from '../constants';

export function useTurnState() {
  const account = useCurrentAccount();
  const client = useCurrentClient();

  const query = useQuery({
    queryKey: ['turnState', TURN_STATE_ID],
    queryFn: async () => {
      const { object } = await client.core.getObject({
        objectId: TURN_STATE_ID,
        include: { json: true },
      });
      const fields = object?.json as Record<string, any>;
      return {
        currentPlayer: Number(fields.current_player),
        turnNumber: Number(fields.turn_number),
        totalPlayers: Number(fields.total_players),
      };
    },
    refetchInterval: 2_000, // Turn changes need faster polling
  });

  const isMyTurn = query.data
    ? account !== null // derive from player index mapping
    : false;

  return { ...query, isMyTurn };
}
```

### Rendering Game State by Phase

```tsx
// src/App.tsx
import { useGameState } from './hooks/useGameState';
import { GameLobby } from './components/game/GameLobby';
import { GameBoard } from './components/game/GameBoard';
import { GameOver } from './components/game/GameOver';

function App() {
  const { data: game, isLoading } = useGameState();

  if (isLoading) return <div>Loading game...</div>;
  if (!game) return <div>Game not found</div>;

  switch (game.state) {
    case 'lobby':
      return <GameLobby game={game} />;
    case 'active':
      return <GameBoard game={game} />;
    case 'finished':
      return <GameOver game={game} />;
  }
}
```

### Multiple Objects in One Query

Batch multiple object reads for efficiency:

```typescript
export function useGameObjects() {
  const client = useCurrentClient();
  return useQuery({
    queryKey: ['gameObjects'],
    queryFn: async () => {
      const [session, grid, turnState] = await Promise.all([
        client.core.getObject({ objectId: GAME_SESSION_ID, include: { json: true } }),
        client.core.getObject({ objectId: GRID_ID, include: { json: true } }),
        client.core.getObject({ objectId: TURN_STATE_ID, include: { json: true } }),
      ]);
      return {
        session: parseGameSession(session.object?.json),
        grid: parseGrid(grid.object?.json),
        turnState: parseTurnState(turnState.object?.json),
      };
    },
    refetchInterval: 3_000,
  });
}
```

### Finding Player Entities

Discover entities owned by the connected wallet:

```typescript
export function useMyEntities(packageId: string) {
  const account = useCurrentAccount();
  const client = useCurrentClient();

  return useQuery({
    queryKey: ['myEntities', account?.address],
    queryFn: async () => {
      const res = await client.core.listOwnedObjects({
        owner: account!.address,
        filter: { StructType: `${packageId}::entity::Entity` },
        include: { json: true },
      });
      return res.objects?.map((obj) =>
        parsePlayerEntity(obj.json as Record<string, any>)
      ) ?? [];
    },
    enabled: !!account,
    refetchInterval: 5_000,
  });
}
```

---

## BCS Struct Deserialization

When on-chain data has nested structs, vectors of structs, or `Option` types, parsed JSON (`showContent: true`) can be unreliable. Use `@mysten/bcs` to define schemas that match your Move structs and deserialize the raw BCS bytes.

### Install

`@mysten/bcs` is already included with `@mysten/sui`. No extra install needed.

### Basic Example

```typescript
import { bcs, fromBase64, type InferBcsType } from '@mysten/bcs';

// Define schema matching your Move struct (field order must match exactly)
const MyStruct = bcs.struct('MyStruct', {
  value: bcs.u64(),
  name: bcs.string(),
  items: bcs.vector(bcs.u64()),         // vector<u64>
  owner: bcs.option(bcs.string()),      // Option<string>
});

// Infer TypeScript type from schema
type MyStructType = InferBcsType<typeof MyStruct>;

// Fetch with showBcs: true (instead of showContent)
const res = await client.core.getObject({
  objectId: '0x...',
  options: { showBcs: true },
});

if (res.data?.bcs?.dataType === 'moveObject') {
  const bytes = fromBase64(res.data.bcs.bcsBytes);
  const parsed: MyStructType = MyStruct.parse(bytes);
  // parsed is fully typed
}
```

### Define Schemas Matching Move Structs

> [!IMPORTANT]
> BCS field order **must exactly match** the Move struct field order. BCS is not self-describing — wrong order = wrong data.

```typescript
// src/lib/bcs-schemas.ts
import { bcs, type InferBcsType } from '@mysten/bcs';

// --- Primitive helpers ---
// UID is a 32-byte address stored as hex
const UID = bcs.fixedArray(32, bcs.u8()).transform({
  input: (id: string) => Array.from(Buffer.from(id.replace('0x', ''), 'hex')),
  output: (bytes) => '0x' + Buffer.from(bytes).toString('hex'),
});

// --- Components (match Move struct field order) ---

const PositionSchema = bcs.struct('Position', {
  x: bcs.u64(),
  y: bcs.u64(),
});

const HealthSchema = bcs.struct('Health', {
  current: bcs.u64(),
  max: bcs.u64(),
});

const EnergySchema = bcs.struct('Energy', {
  current: bcs.u64(),
  max: bcs.u64(),
  regen: bcs.u64(),
});

const AttackSchema = bcs.struct('Attack', {
  value: bcs.u64(),
  range: bcs.u64(),
});

const DefenseSchema = bcs.struct('Defense', {
  armor: bcs.u64(),
  resistance: bcs.u64(),
});

// --- Nested: StatusEffect with vector<Effect> ---

const EffectSchema = bcs.struct('Effect', {
  effect_type: bcs.u8(),
  stacks: bcs.u64(),
  duration: bcs.u8(),
});

const StatusEffectSchema = bcs.struct('StatusEffect', {
  effects: bcs.vector(EffectSchema),
});

// --- Card data (vector of structs) ---

const CardDataSchema = bcs.struct('CardData', {
  card_id: bcs.u64(),
  cost: bcs.u64(),
  attack: bcs.u64(),
  defense: bcs.u64(),
});

const DeckSchema = bcs.struct('Deck', {
  draw_pile: bcs.vector(CardDataSchema),
  hand: bcs.vector(CardDataSchema),
  discard_pile: bcs.vector(CardDataSchema),
});

// --- GameSession with Option<address> ---

const GameSessionSchema = bcs.struct('GameSession', {
  id: UID,
  state: bcs.u8(),
  players: bcs.vector(bcs.fixedArray(32, bcs.u8())),  // vector<address>
  max_players: bcs.u64(),
  winner: bcs.option(bcs.fixedArray(32, bcs.u8())),    // Option<address>
});

// --- Export schemas and inferred types ---

export {
  PositionSchema, HealthSchema, EnergySchema,
  AttackSchema, DefenseSchema, StatusEffectSchema,
  DeckSchema, CardDataSchema, GameSessionSchema,
};

// Infer TypeScript types from schemas
export type Position = InferBcsType<typeof PositionSchema>;
export type Health = InferBcsType<typeof HealthSchema>;
export type Energy = InferBcsType<typeof EnergySchema>;
export type GameSession = InferBcsType<typeof GameSessionSchema>;
export type Deck = InferBcsType<typeof DeckSchema>;
```

### Parse BCS Bytes from Object Data

Use `showBcs: true` instead of `showContent: true` to get raw BCS:

```typescript
// src/hooks/useGameState.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { fromBase64 } from '@mysten/bcs';
import { GAME_SESSION_ID } from '../constants';
import { GameSessionSchema, type GameSession } from '../lib/bcs-schemas';

export function useGameState() {
  const client = useCurrentClient();

  return useQuery<GameSession>({
    queryKey: ['gameSession', GAME_SESSION_ID],
    queryFn: async () => {
      const res = await client.core.getObject({
        objectId: GAME_SESSION_ID,
        options: { showBcs: true },  // ← BCS instead of showContent
      });

      if (res.data?.bcs?.dataType !== 'moveObject') {
        throw new Error('GameSession not found');
      }

      // Parse raw BCS bytes into typed object
      const bytes = fromBase64(res.data.bcs.bcsBytes);
      return GameSessionSchema.parse(bytes);
    },
    refetchInterval: 3_000,
  });
}
```

### When to Use BCS vs Parsed JSON

| Use Case | Approach | Why |
|----------|----------|-----|
| Simple flat structs (u64, string, address) | `showContent: true` + `fields as Record<string, any>` | Simpler, no schema needed |
| Nested structs (Position inside Entity) | `showBcs: true` + BCS schema | Parsed JSON flattens nested structs unpredictably |
| `vector<SomeStruct>` | `showBcs: true` + BCS schema | Parsed JSON loses struct boundaries in vectors |
| `Option<T>` | `showBcs: true` + BCS schema | Parsed JSON uses `{ Some: value }` / `null` inconsistently |
| Dynamic fields | `client.core.listDynamicFields` | Dynamic fields aren't in the parent object's BCS |

### Type Inference

BCS schemas auto-infer TypeScript types — no manual interface definitions needed:

```typescript
import type { InferBcsType } from '@mysten/bcs';

// TS type is automatically: { x: string; y: string }
// (u64 serializes as string in TS)
type Position = InferBcsType<typeof PositionSchema>;

// For input types (more permissive):
// { x: string | number | bigint; y: string | number | bigint }
type PositionInput = typeof PositionSchema.$inferInput;
```
