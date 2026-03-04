# Transaction Patterns

## Creating a Transaction

```typescript
import { Transaction } from '@mysten/sui/transactions';

const tx = new Transaction();
```

## PTB Commands

### `splitCoins` — Split coins from a source

```typescript
const [coin] = tx.splitCoins(tx.gas, [100_000_000]); // split 0.1 SUI from gas
```

### `mergeCoins` — Merge coins into a target

```typescript
tx.mergeCoins(targetCoin, [coin1, coin2]);
```

### `transferObjects` — Transfer objects to an address

```typescript
tx.transferObjects([coin], '0xRecipientAddress');
```

### `moveCall` — Call a Move function

```typescript
tx.moveCall({
  package: '0xPackageId',
  module: 'module_name',
  function: 'function_name',
  typeArguments: ['0x2::sui::SUI'],  // optional type args
  arguments: [
    tx.object('0xObjectId'),          // object reference
    tx.pure.u64(42),                  // typed pure value
    tx.pure.string('hello'),          // string
    tx.pure.address('0xAddress'),     // address
    tx.pure.bool(true),               // boolean
  ],
});
```

### `makeMoveVec` — Create a vector argument

```typescript
const vec = tx.makeMoveVec({
  type: '0x2::coin::Coin<0x2::sui::SUI>',
  elements: [coin1, coin2],
});
```

### `publish` — Publish a package

```typescript
const [upgradeCap] = tx.publish({ modules, dependencies });
```

## Passing Inputs

### JavaScript Values (auto-serialized)

For simple `splitCoins` amounts, JS numbers/bigints work:

```typescript
tx.splitCoins(tx.gas, [100_000_000]); // number auto-serialized
```

### `tx.pure.*` Typed Helpers

```typescript
tx.pure.u8(255)
tx.pure.u16(65535)
tx.pure.u32(4294967295)
tx.pure.u64(1000000000n)
tx.pure.u128(value)
tx.pure.u256(value)
tx.pure.bool(true)
tx.pure.string('hello')
tx.pure.address('0x...')
tx.pure.id('0x...')       // object ID as pure input
tx.pure.option('u64', 42) // Option<u64>
tx.pure.vector('u8', [1, 2, 3])
```

### Object References

```typescript
tx.object('0xObjectId')                        // by ID (auto-resolves mutability)
tx.object.shared('0xObjectId')                 // shared object
tx.object.receiving('0xObjectId')              // receiving object
tx.object.system()                             // 0x5 (Clock) or system objects
```

### BCS Serialization (advanced)

```typescript
import { bcs } from '@mysten/sui/bcs';

tx.pure(bcs.vector(bcs.u64()).serialize([1n, 2n, 3n]));
```

## Composing with `tx.add()` (Codegen Pattern)

When using `@mysten/codegen`, generated functions return `(tx: Transaction) => result`. Use `tx.add()`:

```typescript
import { create as createCounter } from './contracts/counter/counter';

const tx = new Transaction();
tx.add(createCounter());
```

## Signing & Executing

### Via dApp Kit (wallet-connected)

```typescript
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { Transaction } from '@mysten/sui/transactions';

function MyComponent() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function execute() {
    const tx = new Transaction();
    tx.moveCall({ /* ... */ });

    // Sign and execute
    const result = await dAppKit.signAndExecuteTransaction({
      transaction: tx,
    });

    // ALWAYS check for failure
    if (result.FailedTransaction) {
      console.error('Transaction failed:', result.FailedTransaction);
      return;
    }

    // Success — get digest
    const digest = result.Transaction.digest;

    // Wait for finality before reading state
    const txResult = await client.waitForTransaction({
      digest,
      include: { effects: true },
    });

    if (txResult.FailedTransaction) {
      console.error('Transaction failed after waiting');
      return;
    }

    console.log('Effects:', txResult.Transaction.effects);
  }
}
```

### Sign Only (without executing)

```typescript
const signedTx = await dAppKit.signTransaction({ transaction: tx });
// signedTx.bytes, signedTx.signature
```

## Result Discrimination

Transaction results use a discriminated union:

```typescript
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

if (result.FailedTransaction) {
  // result.FailedTransaction contains error info
  throw new Error('Transaction failed');
}

// Success path
const { digest, effects } = result.Transaction;
```

## Extracting Created Objects from Effects

```typescript
const txResult = await client.waitForTransaction({
  digest: result.Transaction.digest,
  include: { effects: true },
});

if (txResult.Transaction) {
  const created = txResult.Transaction.effects?.changedObjects?.find(
    (obj) => obj.idOperation === 'Created',
  );
  console.log('Created object ID:', created?.objectId);
}
```

## `coinWithBalance` Helper

For simple coin transfers without manual `splitCoins`:

```typescript
import { coinWithBalance } from '@mysten/sui/transactions';

const tx = new Transaction();
tx.transferObjects(
  [coinWithBalance({ balance: 100_000_000 })], // 0.1 SUI
  '0xRecipientAddress',
);
```

## Setting Gas Budget

```typescript
tx.setGasBudget(10_000_000); // in MIST
```

## Setting Sender (for sponsored transactions)

```typescript
tx.setSender('0xSponsorAddress');
```

---

## Game Action Patterns

### Core Pattern: Game Action Hook

```
User clicks button → build Transaction → signAndExecuteTransaction → waitForTransaction → refetch state
```

Every game action follows this flow. Wrap it in a custom hook:

```typescript
// src/hooks/useGameActions.ts
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { useQueryClient } from '@tanstack/react-query';
import { PACKAGE_ID, WORLD_ID, GRID_ID, GAME_SESSION_ID, TURN_STATE_ID } from '../constants';

export function useGameActions() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();
  const queryClient = useQueryClient();

  // Shared post-transaction handler
  async function executeAndRefresh(tx: Transaction) {
    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw parseTransactionError(result.FailedTransaction);
    }

    // Wait for finality before reading state.
    // Option A: Use waitForTransaction (preferred when available)
    await client.waitForTransaction({
      digest: result.Transaction.digest,
      include: { effects: true },
    });

    // Option B: Blind delay fallback (if using SuiJsonRpcClient for reads
    // and waitForTransaction is not available on that client)
    // await new Promise(r => setTimeout(r, 2000));

    // Use refetchQueries (not invalidateQueries) for games —
    // it forces an immediate re-fetch rather than waiting for next access
    await Promise.all([
      queryClient.refetchQueries({ queryKey: ['gameSession'] }),
      queryClient.refetchQueries({ queryKey: ['grid'] }),
      queryClient.refetchQueries({ queryKey: ['turnState'] }),
      queryClient.refetchQueries({ queryKey: ['entity'] }),
    ]);

    return result.Transaction;
  }

  return {
    joinGame: () => {
      const tx = new Transaction();
      tx.moveCall({
        package: PACKAGE_ID,
        module: 'game',
        function: 'join_game',
        arguments: [
          tx.object(GAME_SESSION_ID),
          tx.object(WORLD_ID),
          tx.object(GRID_ID),
        ],
      });
      return executeAndRefresh(tx);
    },

    startGame: () => {
      const tx = new Transaction();
      tx.moveCall({
        package: PACKAGE_ID,
        module: 'game',
        function: 'start_game',
        arguments: [tx.object(GAME_SESSION_ID)],
      });
      return executeAndRefresh(tx);
    },

    moveEntity: (entityId: string, x: number, y: number) => {
      const tx = new Transaction();
      tx.moveCall({
        package: PACKAGE_ID,
        module: 'game',
        function: 'move_entity',
        arguments: [
          tx.object(WORLD_ID),
          tx.object(GRID_ID),
          tx.object(entityId),
          tx.pure.u64(x),
          tx.pure.u64(y),
        ],
      });
      return executeAndRefresh(tx);
    },

    attack: (entityId: string, targetId: string) => {
      const tx = new Transaction();
      tx.moveCall({
        package: PACKAGE_ID,
        module: 'game',
        function: 'attack',
        arguments: [
          tx.object(WORLD_ID),
          tx.object(GRID_ID),
          tx.object(entityId),
          tx.object(targetId),
        ],
      });
      return executeAndRefresh(tx);
    },

    endTurn: () => {
      const tx = new Transaction();
      tx.moveCall({
        package: PACKAGE_ID,
        module: 'game',
        function: 'end_turn',
        arguments: [
          tx.object(GAME_SESSION_ID),
          tx.object(TURN_STATE_ID),
        ],
      });
      return executeAndRefresh(tx);
    },
  };
}
```

---

### Handling Move Abort Codes

Move contracts abort with numeric error codes. Map these to user-friendly messages:

```typescript
// src/errors/gameErrors.ts

// Mirror your Move contract's error constants
const GAME_ERROR_MAP: Record<number, string> = {
  101: "It's not your turn",
  102: 'Game is full',
  103: 'Game is not active',
  104: 'Invalid move',
  105: 'Target out of range',
  // Add all error codes from your Move module
};

// Parse the abort code from a failed transaction
export function parseTransactionError(failure: { error: string }): Error {
  const match = failure.error.match(/MoveAbort.*?(\d+)\)?$/);
  if (match) {
    const code = Number(match[1]);
    const message = GAME_ERROR_MAP[code] ?? `Transaction failed (code ${code})`;
    return new Error(message);
  }
  return new Error(failure.error || 'Transaction failed');
}
```

The `ActionBar` component from the usage example below already displays `err.message`, so mapped errors appear automatically.

---

### Batching Multiple Actions (PTB)

Combine move + attack + end turn in one transaction:

```typescript
moveAndAttack: (entityId: string, targetId: string, x: number, y: number) => {
  const tx = new Transaction();

  // Move
  tx.moveCall({
    package: PACKAGE_ID,
    module: 'game',
    function: 'move_entity',
    arguments: [
      tx.object(WORLD_ID), tx.object(GRID_ID),
      tx.object(entityId), tx.pure.u64(x), tx.pure.u64(y),
    ],
  });

  // Attack
  tx.moveCall({
    package: PACKAGE_ID,
    module: 'game',
    function: 'attack',
    arguments: [
      tx.object(WORLD_ID), tx.object(GRID_ID),
      tx.object(entityId), tx.object(targetId),
    ],
  });

  // End turn
  tx.moveCall({
    package: PACKAGE_ID,
    module: 'game',
    function: 'end_turn',
    arguments: [tx.object(GAME_SESSION_ID), tx.object(TURN_STATE_ID)],
  });

  return executeAndRefresh(tx);
},
```

---

### With Codegen Functions

If using `@mysten/codegen`:

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { moveEntity, attack, endTurn } from '../contracts/game/game_module';

const tx = new Transaction();
tx.add(moveEntity({ arguments: [WORLD_ID, GRID_ID, entityId, x, y] }));
tx.add(attack({ arguments: [WORLD_ID, GRID_ID, entityId, targetId] }));
tx.add(endTurn({ arguments: [GAME_SESSION_ID, TURN_STATE_ID] }));
```

---

### Extracting Created Objects (Game-Specific)

When `join_game` creates a player entity, extract its ID:

```typescript
joinGame: async () => {
  const tx = new Transaction();
  tx.moveCall({
    package: PACKAGE_ID,
    module: 'game',
    function: 'join_game',
    arguments: [tx.object(GAME_SESSION_ID), tx.object(WORLD_ID), tx.object(GRID_ID)],
  });

  const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
  if (result.FailedTransaction) throw new Error('Failed to join');

  const txResult = await client.waitForTransaction({
    digest: result.Transaction.digest,
    include: { effects: true },
  });

  if (txResult.FailedTransaction) throw new Error('Failed');

  const created = txResult.Transaction.effects?.changedObjects?.find(
    (obj: any) => obj.idOperation === 'Created',
  );

  return created?.objectId; // The new player entity ID
},
```

---

### Using Actions in Components

```tsx
// src/components/game/ActionBar.tsx
import { useState } from 'react';
import { useGameActions } from '../../hooks/useGameActions';
import { useTurnState } from '../../hooks/useTurnState';

export function ActionBar({ entityId }: { entityId: string }) {
  const { moveEntity, attack, endTurn } = useGameActions();
  const { isMyTurn } = useTurnState();
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleAction(action: () => Promise<any>) {
    setIsPending(true);
    setError(null);
    try {
      await action();
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Action failed');
    } finally {
      setIsPending(false);
    }
  }

  return (
    <div>
      {error && <p className="error">{error}</p>}
      <button
        disabled={!isMyTurn || isPending}
        onClick={() => handleAction(() => moveEntity(entityId, 3, 4))}
      >
        {isPending ? 'Moving...' : 'Move'}
      </button>
      <button
        disabled={!isMyTurn || isPending}
        onClick={() => handleAction(() => endTurn())}
      >
        End Turn
      </button>
    </div>
  );
}
```
