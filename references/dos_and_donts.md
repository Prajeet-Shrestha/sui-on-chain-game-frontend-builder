# Do's and Don'ts

## ❌ Don'ts

### Don't use the legacy `@mysten/dapp-kit`

```typescript
// ❌ WRONG — legacy package
import { ConnectButton } from '@mysten/dapp-kit';

// ✅ CORRECT — new package
import { ConnectButton } from '@mysten/dapp-kit-react/ui';
```

### Don't use the legacy `SuiClient` from `@mysten/sui/client`

```typescript
// ❌ WRONG — legacy, deprecated import path
import { SuiClient } from '@mysten/sui/client';
const client = new SuiClient({ url: '...' });

// ✅ CORRECT — gRPC client (for wallet/tx via dApp Kit)
import { SuiGrpcClient } from '@mysten/sui/grpc';
const client = new SuiGrpcClient({ network: 'testnet', baseUrl: '...' });

// ✅ ALSO CORRECT — JSON-RPC client (for standalone data reads)
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';
const client = new SuiJsonRpcClient({ network: 'testnet', url: getJsonRpcFullnodeUrl('testnet') });
```

> [!NOTE]
> `SuiJsonRpcClient` from `@mysten/sui/jsonRpc` is **not** deprecated. It is the modern JSON-RPC client and is the most reliable option for data reads like `getObject`, `getDynamicFields`, and `multiGetObjects`. See [client_api.md](client_api.md) for the dual-client pattern.

### Don't ignore `FailedTransaction` results

```typescript
// ❌ WRONG — assumes success
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
const digest = result.Transaction.digest; // 💥 crashes if failed

// ✅ CORRECT — check FailedTransaction
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
if (result.FailedTransaction) {
  console.error('Failed:', result.FailedTransaction);
  return;
}
const digest = result.Transaction.digest;
```

### Don't use public endpoints in production

```typescript
// ❌ WRONG for production
baseUrl: 'https://fullnode.mainnet.sui.io:443'

// ✅ CORRECT — use a dedicated RPC provider
baseUrl: 'https://your-provider.example.com'
// e.g., Shinami, BlockVision, Triton, etc.
```

### Don't read state immediately after a transaction

```typescript
// ❌ WRONG — state may not be updated yet
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
const obj = await client.core.getObject({ objectId: '0x...' }); // stale!

// ✅ CORRECT — wait for transaction finality first
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
if (result.FailedTransaction) return;
await client.waitForTransaction({ digest: result.Transaction.digest });
const obj = await client.core.getObject({ objectId: '0x...' }); // fresh!
```

### Don't create standalone clients in components

```typescript
// ❌ WRONG — creates a new client on every render
function Bad() {
  const client = new SuiGrpcClient({ network: 'testnet', baseUrl: '...' });
}

// ✅ CORRECT — use the hook (shares the configured client)
function Good() {
  const client = useCurrentClient();
}
```

### Don't forget the type registration

```typescript
// ❌ WRONG — hooks return generic types (string instead of 'testnet' | 'mainnet')
export const dAppKit = createDAppKit({ ... });
// missing declare module!

// ✅ CORRECT — hooks return correctly narrowed types
export const dAppKit = createDAppKit({ ... });
declare module '@mysten/dapp-kit-react' {
  interface Register { dAppKit: typeof dAppKit; }
}
```

### Don't use raw BCS when `tx.pure.*` helpers exist

```typescript
// ❌ UNNECESSARILY COMPLEX
import { bcs } from '@mysten/sui/bcs';
tx.pure(bcs.u64().serialize(42n));

// ✅ SIMPLER — use typed helpers
tx.pure.u64(42);
```

---

## ✅ Do's

### Always check `result.FailedTransaction` after transactions

```typescript
const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
if (result.FailedTransaction) {
  // Handle error
  return;
}
// Safe to access result.Transaction
```

### Always use `waitForTransaction` before reading post-tx state

```typescript
await client.waitForTransaction({
  digest: result.Transaction.digest,
  include: { effects: true },
});
```

### Use `useCurrentClient()` for data reading in components

It automatically switches when the network changes.

### Use `formatAddress()` for display

```typescript
import { formatAddress } from '@mysten/sui/utils';

// Shows '0x1234...abcd' instead of the full 66-char hex
<p>{formatAddress(account.address)}</p>
```

### Use `tx.pure.*` typed helpers for Move function arguments

```typescript
tx.pure.u64(42)
tx.pure.address('0x...')
tx.pure.string('hello')
tx.pure.bool(true)
tx.pure.vector('u8', [1, 2, 3])
tx.pure.option('u64', 42)
```

### Use `enableBurnerWallet` only in development

```typescript
createDAppKit({
  enableBurnerWallet: import.meta.env.DEV, // Vite
  // enableBurnerWallet: process.env.NODE_ENV === 'development', // Next.js
});
```

### Use `tx.add()` with codegen-generated functions

```typescript
import { create } from './contracts/counter/counter';
tx.add(create());
```

### Handle wallet-not-installed gracefully

```typescript
const wallets = useWallets();
if (wallets.length === 0) {
  // Show install wallet prompt
}
```
