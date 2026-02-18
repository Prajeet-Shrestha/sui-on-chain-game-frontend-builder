# Contract → Frontend Integration

How to read a Move contract and generate the frontend integration layer.

## Step 1: Read the Move Contract

Open the game's `sources/game.move` file and identify these elements:

### 1.1 Structs (→ `lib/types.ts`)

Find all `public struct` declarations with `has key` or `has store`:

```move
public struct GameSession has key, store {
    id: UID,
    state: u8,
    player: address,
    level: u8,
    board: vector<u8>,
    moves_remaining: u64,
}
```

### 1.2 Constants (→ `constants.ts`)

Find all `const` declarations:

```move
const STATE_ACTIVE: u8 = 1;
const STATE_WON: u8 = 2;
const ENotPlayer: u64 = 101;
const EInvalidMove: u64 = 102;
```

### 1.3 Public Functions (→ `hooks/useGameActions.ts`)

Find all `public fun` and `public entry fun` declarations:

```move
public fun start_game(world: &World, level: u8, ctx: &mut TxContext): GameSession
public fun make_move(session: &mut GameSession, direction: u8, ctx: &TxContext)
public entry fun start_and_share(world: &World, level: u8, ctx: &mut TxContext)
```

### 1.4 Events (→ optional, for real-time updates)

Find `public struct ... has copy, drop` (event structs):

```move
public struct GameStarted has copy, drop { session_id: ID, player: address }
```

---

## Step 2: Type Mapping

### Move → TypeScript Type Map

| Move Type | TypeScript Type | Parser |
|-----------|----------------|--------|
| `u8`, `u16`, `u32` | `number` | `Number(field)` |
| `u64`, `u128`, `u256` | `number` | `Number(field)` (or `BigInt` for large values) |
| `bool` | `boolean` | `Boolean(field)` |
| `address` | `string` | direct (already hex string) |
| `ID` | `string` | direct |
| `vector<u8>` | `number[]` | `(field as any[]).map(Number)` |
| `vector<bool>` | `boolean[]` | `(field as any[]).map(Boolean)` |
| `vector<address>` | `string[]` | direct |
| `Option<T>` | `T \| null` | `field?.Some ?? null` |
| `String` / `string::String` | `string` | direct |

---

## Step 3: Generate lib/types.ts

Convert each Move struct to a TypeScript interface:

```typescript
// Move: public struct GameSession has key, store { ... }
export interface GameSession {
    id: string;           // UID → objectId string
    state: 'active' | 'won' | 'lost';  // u8 → mapped enum
    player: string;       // address → string
    level: number;        // u8 → number
    board: number[];      // vector<u8> → number[]
    movesRemaining: number; // u64 → number (camelCase!)
}
```

**Rules:**
- `UID` fields → `id: string` (use `objectId` from query)
- `u8` state fields → TypeScript string union (map via constants)
- Move `snake_case` → TypeScript `camelCase`
- Skip internal-only fields the frontend doesn't need

---

## Step 4: Generate lib/parsers.ts

Create parser functions that convert raw JSON-RPC response fields:

```typescript
import type { GameSession } from './types';
import { STATE_ACTIVE, STATE_WON, STATE_LOST } from '../constants';

function parseState(value: number): 'active' | 'won' | 'lost' {
    if (value === STATE_ACTIVE) return 'active';
    if (value === STATE_WON) return 'won';
    if (value === STATE_LOST) return 'lost';
    return 'active';
}

export function parseGameSession(
    objectId: string,
    fields: Record<string, any>,
): GameSession {
    return {
        id: objectId,
        state: parseState(Number(fields.state)),
        player: fields.player,
        level: Number(fields.level),
        board: (fields.board as any[]).map(Number),
        movesRemaining: Number(fields.moves_remaining),
    };
}
```

**Rules:**
- First argument is always `objectId: string`
- Second argument is `fields: Record<string, any>` from `content.fields`
- Use `Number()` for all numeric fields
- Use `(field as any[]).map(Number)` for `vector<u8>`
- Field names in `fields` use **Move snake_case** (e.g. `fields.moves_remaining`)
- Output properties use **TypeScript camelCase** (e.g. `movesRemaining`)

---

## Step 5: Generate hooks/useGameActions.ts

For each `public fun` / `public entry fun` that modifies state:

1. Identify the **function name** → becomes the hook method name
2. Identify **arguments**:
   - `&World` → `tx.object(WORLD_ID)` (shared constant)
   - `&mut GameSession` → `tx.object(sessionId)` (variable)
   - `u8` → `tx.pure.u8(value)`
   - `u64` → `tx.pure.u64(value)`
   - `&Clock` → `tx.object(CLOCK_ID)`
   - `&TxContext` / `&mut TxContext` → **skip** (auto-injected by runtime)
3. Check return type:
   - Returns an object → capture with `const [result] = tx.moveCall(...)` for PTB chaining
   - Returns void → no capture needed

---

## Step 6: Identify Shared Objects for constants.ts

After `sui client publish`, find shared objects in the output:

```
Created Objects:
  - ID: 0xabc... , Type: 0xpkg::world::World  ← WORLD_ID
  - ID: 0xdef... , Type: 0xpkg::game::Grid   ← GRID_ID
```

Only objects with `has key` that are `share_object()`'d or `transfer::public_share_object()`'d become shared objects you need IDs for.
