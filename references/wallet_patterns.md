# Wallet Connection Patterns

## Pre-Built Components

### `ConnectButton`

All-in-one connect/disconnect button (renders as a Lit web component wrapped for React):

```tsx
import { ConnectButton } from '@mysten/dapp-kit-react/ui';

function Header() {
  return (
    <nav>
      <ConnectButton />
    </nav>
  );
}
```

The `ConnectButton` automatically:
- Shows "Connect Wallet" when disconnected
- Opens a wallet selection modal on click
- Shows connected address + dropdown menu when connected
- Handles disconnect from the dropdown

### `ConnectModal`

Standalone modal for wallet selection:

```tsx
import { ConnectModal } from '@mysten/dapp-kit-react';

function App() {
  return (
    <div>
      <ConnectModal />
      {/* Modal opens programmatically or via ConnectButton */}
    </div>
  );
}
```

## Manual Wallet Connection

### Connect

```typescript
import { useDAppKit, useWallets } from '@mysten/dapp-kit-react';

function ManualConnect() {
  const dAppKit = useDAppKit();
  const wallets = useWallets();

  async function connect(wallet: UiWallet) {
    try {
      const { accounts } = await dAppKit.connectWallet({ wallet });
      console.log('Connected accounts:', accounts);
    } catch (err) {
      console.error('Failed to connect:', err);
    }
  }

  return (
    <ul>
      {wallets.map((wallet) => (
        <li key={wallet.name}>
          <button onClick={() => connect(wallet)}>
            Connect {wallet.name}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### Disconnect

```typescript
const dAppKit = useDAppKit();
await dAppKit.disconnectWallet();
```

### Switch Account

```typescript
const dAppKit = useDAppKit();
dAppKit.switchAccount({ account: targetAccount });
```

### Switch Network

```typescript
const dAppKit = useDAppKit();
dAppKit.switchNetwork('mainnet');
```

## Connection State

### Using `useWalletConnection()`

```typescript
import { useWalletConnection } from '@mysten/dapp-kit-react';

function GatedContent() {
  const { isConnected, isConnecting, account } = useWalletConnection();

  if (isConnecting) return <Spinner />;
  if (!isConnected) return <ConnectButton />;

  return <p>Welcome, {account.address}</p>;
}
```

### Subscribing to Store Changes (Advanced)

For non-React contexts or fine-grained control, subscribe to nanostores directly:

```typescript
const dAppKit = useDAppKit();

// Subscribe to connection changes
const unsubscribe = dAppKit.stores.$connection.subscribe((connection) => {
  console.log('Connection status:', connection.status);
  console.log('Account:', connection.account?.address);
});

// Clean up
unsubscribe();
```

## Signing Personal Messages

For authentication or off-chain verification:

```typescript
const dAppKit = useDAppKit();

async function signMessage() {
  const message = new TextEncoder().encode('Please sign this message');
  
  const { bytes, signature } = await dAppKit.signPersonalMessage({
    message,
  });

  console.log('Signed bytes:', bytes);
  console.log('Signature:', signature);
}
```

## Handling "Wallet Not Installed"

Use `useWallets()` to check available wallets and guide users:

```typescript
function WalletCheck() {
  const wallets = useWallets();

  if (wallets.length === 0) {
    return (
      <div>
        <p>No Sui wallets detected.</p>
        <a href="https://slush.app" target="_blank">Install Slush Wallet</a>
      </div>
    );
  }

  return <ConnectButton />;
}
```

## Auto-Connect Behavior

By default, `createDAppKit` has `autoConnect: true`, which:
1. On app load, checks `localStorage` for last connected wallet
2. Attempts to reconnect silently
3. Sets status to `'reconnecting'` during this process
4. Falls back to `'disconnected'` if reconnection fails

Disable with:
```typescript
const dAppKit = createDAppKit({
  autoConnect: false,
  // ...
});
```
