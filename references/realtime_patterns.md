# Real-Time Update Patterns

## Strategy Overview

On-chain games need fresh state. Choose your approach based on game pace:

| Game Pace | Strategy | `refetchInterval` |
|-----------|----------|-------------------|
| **Turn-based (slow)** | Polling | 3–5 seconds |
| **Turn-based (fast)** | Polling + event hint | 1–2 seconds |
| **Real-time** | Event subscription + polling fallback | 1 second |

---

## 1. Polling with React Query

The simplest and most reliable approach:

```typescript
useQuery({
  queryKey: ['gameSession', GAME_SESSION_ID],
  queryFn: () => fetchGameSession(client),
  refetchInterval: 3_000, // Poll every 3 seconds
  refetchIntervalInBackground: false, // Stop when tab is hidden
});
```

### Adaptive Polling

Poll faster during active gameplay, slower in lobby:

```typescript
export function useGameState() {
  const client = useCurrentClient();
  const [gamePhase, setGamePhase] = useState<'lobby' | 'active' | 'finished'>('lobby');

  const pollInterval = {
    lobby: 5_000,     // Check for new players every 5s
    active: 2_000,    // Active game — every 2s
    finished: 0,      // Stop polling when game is over
  }[gamePhase];

  return useQuery({
    queryKey: ['gameSession', GAME_SESSION_ID],
    queryFn: async () => {
      const game = await fetchGameSession(client);
      setGamePhase(game.state);
      return game;
    },
    refetchInterval: pollInterval || false,
  });
}
```

---

## 2. Optimistic Updates

Update the UI immediately, then confirm with on-chain data:

```typescript
import { useQueryClient } from '@tanstack/react-query';

function useOptimisticMove() {
  const queryClient = useQueryClient();
  const { moveEntity } = useGameActions();

  async function optimisticMove(entityId: string, newX: number, newY: number) {
    // 1. Optimistically update the grid
    queryClient.setQueryData(['grid', GRID_ID], (old: Grid | undefined) => {
      if (!old) return old;
      return {
        ...old,
        cells: old.cells.map((cell) => {
          if (cell.occupant === entityId) return { ...cell, occupant: null };
          if (cell.x === newX && cell.y === newY) return { ...cell, occupant: entityId };
          return cell;
        }),
      };
    });

    try {
      // 2. Execute the real transaction
      await moveEntity(entityId, newX, newY);
    } catch (err) {
      // 3. Rollback on failure — refetch real state
      queryClient.invalidateQueries({ queryKey: ['grid'] });
      throw err;
    }
  }

  return optimisticMove;
}
```

---

## 3. Post-Transaction Refresh

Always refetch after your own transaction succeeds:

```typescript
async function executeAndRefresh(tx: Transaction) {
  const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
  if (result.FailedTransaction) throw new Error('Failed');

  await client.waitForTransaction({ digest: result.Transaction.digest });

  // Use refetchQueries for games — forces immediate re-fetch
  await Promise.all([
    queryClient.refetchQueries({ queryKey: ['gameSession'] }),
    queryClient.refetchQueries({ queryKey: ['grid'] }),
    queryClient.refetchQueries({ queryKey: ['turnState'] }),
    queryClient.refetchQueries({ queryKey: ['entity'] }),
  ]);
}
```

> [!TIP]
> **Blind-delay fallback:** If you're using `SuiJsonRpcClient` for data reads and `waitForTransaction` isn't available on it, replace the `waitForTransaction` call with `await new Promise(r => setTimeout(r, 2000))`. This is less precise (wastes time on fast days, may fail on slow indexer days) but is a pragmatic workaround. See [game_transactions.md](game_transactions.md) for the full pattern.

---

## 4. Game Events (Activity Log & Notifications)

> [!IMPORTANT]
> **Object polling is the primary state source** (sections 1–3 above).
> Events are for **activity logs** and **UI notifications**, not for reading current game state.
> Sui does not support WebSocket event subscriptions — events are polled.

### Define TypeScript Types Matching Move Event Structs

```typescript
// src/lib/events.ts
import { PACKAGE_ID } from '../constants';

// Match your Move event structs exactly
interface GameCreatedEvent {
  game_id: string;
  creator: string;
}

interface PlayerJoinedEvent {
  game_id: string;
  player: string;
  player_index: string; // u64 comes as string
}

interface GameStartedEvent {
  game_id: string;
}

interface GameOverEvent {
  game_id: string;
  winner: string;
}

// Discriminated union for type-safe handling
type GameEvent =
  | { type: 'GameCreated'; data: GameCreatedEvent; timestamp: string }
  | { type: 'PlayerJoined'; data: PlayerJoinedEvent; timestamp: string }
  | { type: 'GameStarted'; data: GameStartedEvent; timestamp: string }
  | { type: 'GameOver'; data: GameOverEvent; timestamp: string };

// Map Move type string → short name
// event.type looks like "0xPKG::game::GameCreated"
function parseEventType(typeRepr: string): string | null {
  const match = typeRepr.match(/::(\w+)$/);
  return match ? match[1] : null;
}

// Parse a raw event node into a typed GameEvent
export function parseGameEvent(node: {
  type: { repr: string };
  json: any;
  timestamp: string;
}): GameEvent | null {
  const name = parseEventType(node.type.repr);
  if (!name) return null;

  switch (name) {
    case 'GameCreated':
      return { type: 'GameCreated', data: node.json, timestamp: node.timestamp };
    case 'PlayerJoined':
      return { type: 'PlayerJoined', data: node.json, timestamp: node.timestamp };
    case 'GameStarted':
      return { type: 'GameStarted', data: node.json, timestamp: node.timestamp };
    case 'GameOver':
      return { type: 'GameOver', data: node.json, timestamp: node.timestamp };
    default:
      return null;
  }
}
```

### Poll Events for Activity Log

```typescript
// src/hooks/useGameLog.ts
import { useQuery } from '@tanstack/react-query';
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { PACKAGE_ID } from '../constants';
import { parseGameEvent, type GameEvent } from '../lib/events';

export function useGameLog() {
  const client = useCurrentClient();

  return useQuery<GameEvent[]>({
    queryKey: ['gameLog', PACKAGE_ID],
    queryFn: async () => {
      const res = await client.core.queryEvents({
        query: { MoveModule: { package: PACKAGE_ID, module: 'game' } },
        order: 'descending',
        limit: 50,
      });

      return (res.data ?? [])
        .map((event) => parseGameEvent({
          type: { repr: event.type },
          json: event.parsedJson,
          timestamp: event.timestampMs ?? '',
        }))
        .filter((e): e is GameEvent => e !== null);
    },
    refetchInterval: 5_000, // Activity log doesn't need fast polling
  });
}
```

### Render Activity Log

```tsx
// src/components/game/GameLog.tsx
import { useGameLog } from '../../hooks/useGameLog';
import { formatAddress } from '@mysten/sui/utils';

function eventToMessage(event: GameEvent): string {
  switch (event.type) {
    case 'GameCreated':
      return `Game created by ${formatAddress(event.data.creator)}`;
    case 'PlayerJoined':
      return `${formatAddress(event.data.player)} joined as Player ${Number(event.data.player_index) + 1}`;
    case 'GameStarted':
      return 'Game started!';
    case 'GameOver':
      return `${formatAddress(event.data.winner)} wins!`;
  }
}

export function GameLog() {
  const { data: events } = useGameLog();

  return (
    <div className="game-log">
      <h3>Activity Log</h3>
      <ul>
        {events?.map((event, i) => (
          <li key={i}>{eventToMessage(event)}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Event-Driven Notifications

Use events to trigger toasts or sounds without replacing object polling:

```typescript
// src/hooks/useGameNotifications.ts
import { useEffect, useRef } from 'react';
import { useGameLog } from './useGameLog';
import { useCurrentAccount } from '@mysten/dapp-kit-react';

export function useGameNotifications() {
  const { data: events } = useGameLog();
  const account = useCurrentAccount();
  const lastSeenCount = useRef(0);

  useEffect(() => {
    if (!events || events.length <= lastSeenCount.current) return;

    // Only process new events
    const newEvents = events.slice(0, events.length - lastSeenCount.current);
    lastSeenCount.current = events.length;

    for (const event of newEvents) {
      switch (event.type) {
        case 'PlayerJoined':
          if (event.data.player !== account?.address) {
            showToast(`A new player joined!`);
          }
          break;
        case 'GameStarted':
          showToast('Game started — good luck!');
          break;
        case 'GameOver':
          if (event.data.winner === account?.address) {
            showToast('🎉 You won!');
          } else {
            showToast('Game over — better luck next time');
          }
          break;
      }
    }
  }, [events, account]);
}
```
```

---

## 5. Turn Change Detection

Detect when it becomes the player's turn:

```typescript
import { useEffect, useRef } from 'react';

function useTurnNotification() {
  const { data: turnState, isMyTurn } = useTurnState();
  const prevIsMyTurn = useRef(false);

  useEffect(() => {
    if (isMyTurn && !prevIsMyTurn.current) {
      // Just became my turn!
      playSound('your-turn.mp3');
      showNotification('Your turn!');
    }
    prevIsMyTurn.current = isMyTurn;
  }, [isMyTurn]);
}
```

---

## 6. Stale Data Prevention

Don't render stale data as if it's fresh:

```tsx
function GameBoard() {
  const { data: game, dataUpdatedAt, isFetching } = useGameState();

  const staleness = Date.now() - dataUpdatedAt;
  const isStale = staleness > 10_000; // 10 seconds

  return (
    <div>
      {isFetching && <div className="loading-indicator">Refreshing...</div>}
      {isStale && <div className="stale-warning">Data may be outdated</div>}
      {/* Render game board */}
    </div>
  );
}
```
