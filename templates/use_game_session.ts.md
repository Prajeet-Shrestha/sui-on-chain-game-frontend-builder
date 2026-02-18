# Template: hooks/useGameSession.ts

React Query hook for polling on-chain game state. Pattern is consistent across all 8 games.

```typescript
import { useQuery } from '@tanstack/react-query';
import { suiClient } from '../lib/suiClient';
import { parseGameSession } from '../lib/parsers';
import type { GameSession } from '../lib/types';

export function useGameSession(sessionId: string | null) {
    return useQuery<GameSession>({
        queryKey: ['gameSession', sessionId],
        queryFn: async () => {
            const res = await suiClient.getObject({
                id: sessionId!,
                options: { showContent: true },
            });
            if (res.data?.content?.dataType !== 'moveObject') {
                throw new Error('GameSession not found');
            }
            return parseGameSession(
                res.data.objectId,
                res.data.content.fields as Record<string, any>,
            );
        },
        enabled: !!sessionId,
        refetchInterval: 2_000, // poll every 2s for fresh state
    });
}
```

## Variations seen in games

### Multiple queries (sokoban, tactics_ogre)
Some games need to query multiple objects:
```typescript
export function useGameState(sessionId: string | null) {
    const session = useGameSession(sessionId);
    const grid = useGrid(session.data?.gridId ?? null);
    const entities = useEntities(session.data?.entityIds ?? []);
    return { session, grid, entities };
}
```

### Owned objects (card_crawler)
Query the connected player's owned game objects:
```typescript
import { useCurrentAccount } from '@mysten/dapp-kit-react';

export function usePlayerGames() {
    const account = useCurrentAccount();
    return useQuery({
        queryKey: ['playerGames', account?.address],
        queryFn: async () => {
            const res = await suiClient.getOwnedObjects({
                owner: account!.address,
                filter: { StructType: `${PACKAGE_ID}::game::GameSession` },
                options: { showContent: true },
            });
            return res.data.map(obj => 
                parseGameSession(obj.data!.objectId, obj.data!.content!.fields as any)
            );
        },
        enabled: !!account,
    });
}
```
