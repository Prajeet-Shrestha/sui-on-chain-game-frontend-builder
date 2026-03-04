# Template: hooks/useGameActions.ts

The core hook for executing game transactions. Extracted from all 8 games with key variations.

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { PACKAGE_ID, WORLD_ID, GAME_ERROR_MAP } from '../constants';
import { useUIStore } from '../stores/uiStore';

// ─── Error Parser ───────────────────────────────
// Extracts Move abort codes and maps to human-readable messages

function parseTransactionError(failure: { error: string }): Error {
    const errorStr = typeof failure.error === 'string'
        ? failure.error
        : JSON.stringify(failure.error);
    const match = errorStr.match(/MoveAbort.*?(\d+)\)?$/);
    if (match) {
        const code = Number(match[1]);
        const message = GAME_ERROR_MAP[code] ?? `Transaction failed (code ${code})`;
        return new Error(message);
    }
    return new Error(errorStr || 'Transaction failed');
}

// ─── Hook ───────────────────────────────────────

export function useGameActions() {
    const dAppKit = useDAppKit();
    const client = useCurrentClient();
    const { setIsPending, setError } = useUIStore();

    // Wraps any PTB execution with pending/error state management
    async function execute(tx: Transaction) {
        setIsPending(true);
        setError(null);
        try {
            const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

            if (result.FailedTransaction) {
                throw parseTransactionError(result.FailedTransaction as any);
            }

            await client.waitForTransaction({
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
        // Replace with your actual contract entry functions:
        //
        // startGame: (level: number) => {
        //     const tx = new Transaction();
        //     tx.moveCall({
        //         package: PACKAGE_ID,
        //         module: 'game',
        //         function: 'start_game',
        //         arguments: [
        //             tx.object(WORLD_ID),
        //             tx.pure.u64(level),
        //         ],
        //     });
        //     return execute(tx);
        // },
    };
}
```

## Real-World Patterns

### Pattern A: Discover created objects from tx result (sokoban, tactics_ogre, flappy_bird)

When a transaction creates new shared objects, find them in the result:

```typescript
startLevel: async (levelId: number) => {
    const tx = new Transaction();
    tx.moveCall({ ... });

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
    if (result.FailedTransaction) {
        throw parseTransactionError(result.FailedTransaction as any);
    }

    await client.waitForTransaction({
        digest: result.Transaction.digest,
    });

    // Fetch tx details to find created objects
    const txResponse = await suiClient.getTransactionBlock({
        digest: result.Transaction.digest,
        options: { showEvents: true, showObjectChanges: true },
    });

    // Find by event (preferred — e.g. LevelStarted event)
    const event = txResponse.events?.find(e =>
        e.type?.includes('::game::LevelStarted')
    );
    const sessionId = event?.parsedJson?.session_id;

    // Fallback: find by objectChanges type
    const created = txResponse.objectChanges?.filter(c => c.type === 'created');
    const sessionObj = created?.find(c => c.objectType?.includes('GameSession'));
},
```

### Pattern B: Refetch React Query caches post-transaction (tactics_ogre)

```typescript
import { useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

async function executeAndRefresh(tx: Transaction, extraKeys: string[] = []) {
    const result = await execute(tx);
    // Invalidate relevant caches
    const keys = ['gameSession', 'grid', 'entities', ...extraKeys];
    await Promise.all(keys.map(k => queryClient.refetchQueries({ queryKey: [k] })));
    return result;
}
```

### Pattern C: Multi-step PTB (virus_game)

Chain multiple move calls atomically in one transaction:

```typescript
const [session] = tx.moveCall({ ...startLevel });
for (const color of colors) {
    tx.moveCall({ ...chooseColor, arguments: [session, tx.pure.u8(color)] });
}
tx.moveCall({ ...shareSession, arguments: [session] });
```

### Pattern D: Pass vectors (sokoban)

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

### Pattern E: Auto-fund with faucet (flappy_bird)

```typescript
import { requestSuiFromFaucetV2, getFaucetHost } from '@mysten/sui/faucet';

async function ensureGas(address: string) {
    const balance = await suiClient.getBalance({ owner: address });
    if (BigInt(balance?.totalBalance ?? '0') < 10_000_000n) {
        await requestSuiFromFaucetV2({
            host: getFaucetHost('testnet'),
            recipient: address,
        });
        await new Promise(r => setTimeout(r, 1500)); // wait for funding
    }
}
```
