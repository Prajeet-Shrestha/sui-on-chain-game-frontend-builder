# Project Setup & Provider Configuration

## 1. Scaffold with Vite

```bash
npm create vite@latest my-game -- --template react-ts
cd my-game
```

## 2. Install Dependencies

```bash
npm i @mysten/dapp-kit-react @mysten/sui @tanstack/react-query
```

Optional but recommended:
```bash
npm i zustand                    # Client-side game state management
npm i framer-motion              # Smooth animations
npm i lucide-react               # Icons
```

> `@mysten/dapp-kit-react` re-exports everything from `@mysten/dapp-kit-core`, so you only need the packages above.

## 3. Create the dApp Kit Instance

Create a file `src/dApp-kit.ts`:

```typescript
import { createDAppKit } from '@mysten/dapp-kit-react';
import { SuiGrpcClient } from '@mysten/sui/grpc';

const GRPC_URLS = {
  mainnet: 'https://fullnode.mainnet.sui.io:443',
  testnet: 'https://fullnode.testnet.sui.io:443',
  devnet: 'https://fullnode.devnet.sui.io:443',
} as const;

export const dAppKit = createDAppKit({
  networks: ['testnet', 'devnet'],
  defaultNetwork: 'testnet',
  enableBurnerWallet: import.meta.env.DEV, // dev-only burner wallet
  createClient(network) {
    return new SuiGrpcClient({ network, baseUrl: GRPC_URLS[network] });
  },
});

// REQUIRED: Global type registration for hook inference
declare module '@mysten/dapp-kit-react' {
  interface Register {
    dAppKit: typeof dAppKit;
  }
}
```

### `createDAppKit` Options

| Option | Default | Description |
|--------|---------|-------------|
| `networks` | (required) | Array of network name strings |
| `createClient(network)` | (required) | Factory returning a `DAppKitCompatibleClient` |
| `defaultNetwork` | `networks[0]` | Initial active network |
| `autoConnect` | `true` | Auto-reconnect to last used wallet |
| `enableBurnerWallet` | `false` | Enable dev-only burner wallet |
| `slushWalletConfig` | auto | Slush web wallet config, `null` to disable |
| `storage` | `localStorage` | Where to persist last connected wallet |
| `storageKey` | `mysten-dapp-kit:selected-wallet-and-address` | Storage key |
| `walletInitializers` | `[]` | Additional wallet standard wallet initializers |

## 4. Create Standalone Data Reader

Create `src/lib/suiClient.ts` for data reads (see [client_api.md](client_api.md#suijsonrpcclient) for why):

```typescript
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

export const suiClient = new SuiJsonRpcClient({
  network: 'testnet',
  url: getJsonRpcFullnodeUrl('testnet'),
});
```

## 5. Configure Entry Point

`src/main.tsx`:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { DAppKitProvider } from '@mysten/dapp-kit-react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { dAppKit } from './dApp-kit';
import App from './App';
import './index.css';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Games need fresh data — shorter stale time
      staleTime: 2_000,
      refetchOnWindowFocus: true,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <DAppKitProvider dAppKit={dAppKit}>
        <App />
      </DAppKitProvider>
    </QueryClientProvider>
  </React.StrictMode>,
);
```

## 6. Game Constants

Create `src/constants.ts`:

```typescript
// Update these after deploying your game contract
export const PACKAGE_ID = '0xYourPackageId';
export const WORLD_ID = '0xYourWorldObjectId';
export const GRID_ID = '0xYourGridObjectId';
export const GAME_SESSION_ID = '0xYourGameSessionId';
export const TURN_STATE_ID = '0xYourTurnStateId';

// Per-network overrides (optional)
export const PACKAGE_IDS: Record<string, string> = {
  testnet: '0x...',
  devnet: '0x...',
};
```

## 7. Network Endpoints

| Network | gRPC Base URL |
|---------|--------------|
| Mainnet | `https://fullnode.mainnet.sui.io:443` |
| Testnet | `https://fullnode.testnet.sui.io:443` |
| Devnet | `https://fullnode.devnet.sui.io:443` |
| Localnet | `http://127.0.0.1:9000` |

> **⚠️ Production Warning:** The public `fullnode.*.sui.io` endpoints are rate-limited. Use a dedicated RPC provider (e.g., Shinami, BlockVision, Triton) for production apps.

## 8. Vite Config (Optional Tweaks)

`vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': '/src',
    },
  },
  build: {
    target: 'esnext', // gRPC needs modern JS
  },
});
```

## 9. Type Registration (Why It Matters)

The `declare module` block augments the `Register` interface so that all hooks (`useDAppKit`, `useCurrentClient`, `useCurrentNetwork`, etc.) return **correctly typed** values matching your specific `createDAppKit` configuration. Without it, hooks return generic types.

```typescript
// This makes useCurrentNetwork() return 'mainnet' | 'testnet' | 'devnet'
// instead of just `string`
declare module '@mysten/dapp-kit-react' {
  interface Register {
    dAppKit: typeof dAppKit;
  }
}
```

## 10. MVR (Move Version Registry) Overrides

For projects using `@mysten/codegen`, you can map local package names to deployed addresses per network:

```typescript
export const dAppKit = createDAppKit({
  networks: ['testnet', 'mainnet'],
  createClient(network) {
    return new SuiGrpcClient({
      network,
      baseUrl: GRPC_URLS[network],
      mvr: {
        overrides: {
          packages: {
            '@local-pkg/mycontract': PACKAGE_IDS[network],
          },
        },
      },
    });
  },
});
```

## 11. Dev Server

```bash
npm run dev
# Open http://localhost:5173
```
