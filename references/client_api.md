# Client API Reference

## Available Clients

| Client | Import | Use Case |
|--------|--------|----------|
| `SuiGrpcClient` | `@mysten/sui/grpc` | **Recommended** for wallet/tx via dApp Kit. May have URL resolver issues for standalone data reads |
| `SuiJsonRpcClient` | `@mysten/sui/jsonRpc` | **Reliable** for data reads (`getObject`, `getDynamicFields`, `multiGetObjects`). Use as standalone client alongside dApp Kit |
| `SuiGraphQLClient` | `@mysten/sui/graphql` | Complex/filtered queries, custom selections |

> [!WARNING]
> The legacy `SuiClient` from `@mysten/sui/client` is deprecated. Use `SuiJsonRpcClient` from `@mysten/sui/jsonRpc` instead — it is the modern JSON-RPC client and is **not** deprecated.

## `SuiGrpcClient`

### Constructor

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';

const client = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
});
```

### With MVR Overrides (for codegen)

```typescript
const client = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
  mvr: {
    overrides: {
      packages: {
        '@local-pkg/mycontract': '0xDeployedPackageId',
      },
    },
  },
});
```

### Core API (`client.core.*`)

The Core API is a transport-agnostic interface shared across all client types:

```typescript
// Objects
await client.core.getObject({ objectId, options });
await client.core.multiGetObjects({ ids, options });
await client.core.getOwnedObjects({ owner, filter, options });

// Balances
await client.core.getBalance({ owner, coinType });
await client.core.getAllBalances({ owner });
await client.core.getCoins({ owner, coinType, cursor, limit });

// Transactions
await client.core.getTransactionBlock({ digest, options });
await client.core.queryTransactionBlocks({ filter, options, cursor, limit });

// Events
await client.core.queryEvents({ query, cursor, limit });

// System
await client.core.getReferenceGasPrice();
await client.core.getLatestSuiSystemState();
await client.core.getChainIdentifier();
```

### gRPC Service Clients

Direct access to gRPC services for lower-level queries:

```typescript
// State service — objects, dynamic fields
const objects = await client.stateService.listOwnedObjects({
  owner: '0x...',
});

// Ledger service — transactions, checkpoints
const tx = await client.ledgerService.getTransaction({
  digest: '...',
});

// Execute service — dry run, execute
const result = await client.executeService.executeTransaction({
  transaction: txBytes,
  signatures: [signature],
});
```

### Wait for Transaction

```typescript
const result = await client.waitForTransaction({
  digest: 'DigestString',
  include: {
    effects: true,
    events: true,
    // balanceChanges: true,
    // objectChanges: true,
  },
});
```

## `SuiGraphQLClient`

For complex queries with custom field selection:

### Setup

```typescript
import { SuiGraphQLClient } from '@mysten/sui/graphql';
import { graphql } from '@mysten/sui/graphql/schemas/latest';

const gqlClient = new SuiGraphQLClient({
  url: 'https://sui-testnet.mystenlabs.com/graphql',
});
```

### Typed Queries

```typescript
const BalanceQuery = graphql(`
  query GetBalance($address: SuiAddress!) {
    address(address: $address) {
      balance {
        totalBalance
      }
    }
  }
`);

const result = await gqlClient.execute(BalanceQuery, {
  address: '0x...',
});
```

### When to Use GraphQL

- Complex nested queries (e.g., objects with dynamic fields with nested objects)
- Custom field selection to minimize response size
- Filtering by multiple criteria simultaneously
- Aggregation queries

## `SuiJsonRpcClient`

Reliable JSON-RPC client for data reads. Use this when `SuiGrpcClient` has URL resolver issues, or when you need `getDynamicFields` for ECS entity components.

### Constructor

```typescript
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

const client = new SuiJsonRpcClient({
  network: 'testnet',
  url: getJsonRpcFullnodeUrl('testnet'),
});
```

### `getJsonRpcFullnodeUrl` Helper

Returns the public fullnode URL for a given network:

```typescript
getJsonRpcFullnodeUrl('mainnet')  // 'https://fullnode.mainnet.sui.io:443'
getJsonRpcFullnodeUrl('testnet')  // 'https://fullnode.testnet.sui.io:443'
getJsonRpcFullnodeUrl('devnet')   // 'https://fullnode.devnet.sui.io:443'
```

### API Surface

The JSON-RPC client exposes methods directly (no `.core` namespace):

```typescript
await client.getObject({ id: '0x...', options: { showContent: true } });
await client.multiGetObjects({ ids: ['0x...'], options: { showContent: true } });
await client.getDynamicFields({ parentId: '0x...' });
await client.getOwnedObjects({ owner: '0x...' });
await client.getBalance({ owner: '0x...' });
```

> [!NOTE]
> The JSON-RPC client uses `id` for single object lookups, while the gRPC Core API uses `objectId`. Both are correct for their respective clients.

### Recommended: Dual-Client Pattern

Use gRPC via dApp Kit for wallet connection and transactions, and a standalone JSON-RPC client for data reads:

```typescript
// src/lib/suiClient.ts — standalone data reader
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

export const suiClient = new SuiJsonRpcClient({
  network: 'testnet',
  url: getJsonRpcFullnodeUrl('testnet'),
});

// src/dApp-kit.ts — wallet/tx via gRPC
import { createDAppKit } from '@mysten/dapp-kit-react';
import { SuiGrpcClient } from '@mysten/sui/grpc';

export const dAppKit = createDAppKit({
  networks: ['testnet', 'devnet'],
  createClient(network) {
    return new SuiGrpcClient({ network });
  },
});
```

## Using `useCurrentClient()` in Components

The `useCurrentClient()` hook returns the client for the active network. It updates automatically when the user switches networks:

```typescript
import { useCurrentClient } from '@mysten/dapp-kit-react';

function NetworkAwareComponent() {
  const client = useCurrentClient();

  // client automatically points to the correct network
  // Re-renders when network changes
}
```

## Getting a Client for a Specific Network

```typescript
const dAppKit = useDAppKit();

// Get client for a specific network, regardless of current selection
const mainnetClient = dAppKit.getClient('mainnet');
const testnetClient = dAppKit.getClient('testnet');

// Get client for current network
const currentClient = dAppKit.getClient();
```
