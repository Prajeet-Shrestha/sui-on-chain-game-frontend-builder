# React Hooks API Reference

All hooks are imported from `@mysten/dapp-kit-react`. Every hook accepts an optional `{ dAppKit }` parameter to use a specific instance (defaults to the one from `DAppKitProvider` context).

---

## `useDAppKit()`

Returns the full `DAppKit` instance with all actions and stores.

```typescript
import { useDAppKit } from '@mysten/dapp-kit-react';

function MyComponent() {
  const dAppKit = useDAppKit();

  // Actions
  await dAppKit.connectWallet({ wallet });
  await dAppKit.disconnectWallet();
  await dAppKit.signAndExecuteTransaction({ transaction: tx });
  await dAppKit.signTransaction({ transaction: tx });
  await dAppKit.signPersonalMessage({ message: new Uint8Array([...]) });
  dAppKit.switchAccount({ account });
  dAppKit.switchNetwork('mainnet');

  // Getters
  const client = dAppKit.getClient();          // current network client
  const client2 = dAppKit.getClient('testnet'); // specific network client

  // Stores (nanostores — use useStore() or other hooks)
  dAppKit.stores.$wallets;         // compatible wallets
  dAppKit.stores.$connection;      // wallet connection state
  dAppKit.stores.$currentNetwork;  // active network
  dAppKit.stores.$currentClient;   // active client
}
```

### Return Type: `DAppKit<TNetworks, Client>`

| Property | Type | Description |
|----------|------|-------------|
| `connectWallet(args)` | `Promise<{ accounts }>` | Connect to a wallet |
| `disconnectWallet()` | `Promise<void>` | Disconnect current wallet |
| `signTransaction(args)` | `Promise<SignedTransaction>` | Sign without executing |
| `signAndExecuteTransaction(args)` | `Promise<Result>` | Sign + execute, returns `{ $kind, ... }` |
| `signPersonalMessage(args)` | `Promise<SignedPersonalMessage>` | Sign an arbitrary message |
| `switchAccount(args)` | `void` | Switch to a different account |
| `switchNetwork(network)` | `void` | Switch active network |
| `getClient(network?)` | `Client` | Get client for network (default: current) |
| `networks` | `TNetworks` | Configured network array |
| `stores` | `{ $wallets, $connection, $currentNetwork, $currentClient }` | Nanostore atoms |

---

## `useCurrentAccount()`

Returns the currently connected wallet account, or `null` if disconnected.

```typescript
import { useCurrentAccount } from '@mysten/dapp-kit-react';

function Profile() {
  const account = useCurrentAccount();

  if (!account) return <p>Not connected</p>;
  return <p>Address: {account.address}</p>;
}
```

**Returns:** `UiWalletAccount | null`

Key fields on `UiWalletAccount`:
- `address` — the Sui address string
- `publicKey` — public key bytes
- `chains` — supported chain identifiers
- `features` — supported wallet features

---

## `useCurrentWallet()`

Returns the currently connected wallet, or `null`.

```typescript
import { useCurrentWallet } from '@mysten/dapp-kit-react';

function WalletInfo() {
  const wallet = useCurrentWallet();

  if (!wallet) return <p>No wallet</p>;
  return <p>Wallet: {wallet.name}</p>;
}
```

**Returns:** `UiWallet | null`

Key fields on `UiWallet`:
- `name` — wallet display name
- `icon` — wallet icon URL
- `chains` — supported chains
- `features` — supported features
- `accounts` — array of `UiWalletAccount`

---

## `useCurrentNetwork()`

Returns the currently active network string.

```typescript
import { useCurrentNetwork } from '@mysten/dapp-kit-react';

function NetworkBadge() {
  const network = useCurrentNetwork();
  return <span>Network: {network}</span>; // e.g. 'testnet'
}
```

**Returns:** `TNetworks[number]` (e.g., `'mainnet' | 'testnet' | 'devnet'` if type registration is done)

---

## `useCurrentClient()`

Returns the Sui client instance for the currently active network.

```typescript
import { useCurrentClient } from '@mysten/dapp-kit-react';

function DataFetcher() {
  const client = useCurrentClient();

  useEffect(() => {
    client.core.getBalance({ owner: '0x...' }).then(console.log);
  }, [client]);
}
```

**Returns:** The client type returned by your `createClient` function (typically `SuiGrpcClient`).

---

## `useWalletConnection()`

Returns the full connection state object with status and boolean convenience flags.

```typescript
import { useWalletConnection } from '@mysten/dapp-kit-react';

function ConnectionStatus() {
  const connection = useWalletConnection();

  // connection.status: 'connected' | 'connecting' | 'reconnecting' | 'disconnected'
  // connection.wallet: UiWallet | null
  // connection.account: UiWalletAccount | null
  // connection.supportedIntents: string[]

  // Boolean flags:
  // connection.isConnected
  // connection.isConnecting
  // connection.isReconnecting
  // connection.isDisconnected

  if (connection.isConnecting) return <p>Connecting...</p>;
  if (connection.isConnected) return <p>Connected: {connection.account.address}</p>;
  return <p>Disconnected</p>;
}
```

### `WalletConnection` States

| Status | `wallet` | `account` | Boolean Flags |
|--------|----------|-----------|---------------|
| `'disconnected'` | `null` | `null` | `isDisconnected: true` |
| `'connecting'` | `null` | `null` | `isConnecting: true` |
| `'reconnecting'` | `UiWallet` | `UiWalletAccount` | `isReconnecting: true` |
| `'connected'` | `UiWallet` | `UiWalletAccount` | `isConnected: true` |

---

## `useWallets()`

Returns the list of all detected compatible wallets.

```typescript
import { useWallets } from '@mysten/dapp-kit-react';

function WalletPicker() {
  const wallets = useWallets();

  return (
    <ul>
      {wallets.map((wallet) => (
        <li key={wallet.name}>
          <img src={wallet.icon} alt={wallet.name} />
          {wallet.name}
        </li>
      ))}
    </ul>
  );
}
```

**Returns:** `UiWallet[]` — only wallets compatible with the current network and having required features.
