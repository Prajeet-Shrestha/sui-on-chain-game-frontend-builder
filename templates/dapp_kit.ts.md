# Template: dApp-kit.ts

Sui dApp Kit configuration + type registration. Two variants exist across games.

## Recommended (with explicit GRPC URLs)

Used by: virus_game, sokoban, flappy_bird, maze_game, tactics_ogre

```typescript
import { createDAppKit } from '@mysten/dapp-kit-core';
import { SuiGrpcClient } from '@mysten/sui/grpc';

const GRPC_URLS: Record<string, string> = {
    testnet: 'https://fullnode.testnet.sui.io:443',
    devnet: 'https://fullnode.devnet.sui.io:443',
};

export const dAppKit = createDAppKit({
    networks: ['testnet', 'devnet'],
    defaultNetwork: 'testnet',
    enableBurnerWallet: import.meta.env.DEV,
    createClient(network) {
        return new SuiGrpcClient({
            network,
            baseUrl: GRPC_URLS[network] ?? GRPC_URLS.testnet,
        });
    },
});

// REQUIRED — enables typed hooks (useCurrentNetwork returns 'testnet' | 'devnet')
declare module '@mysten/dapp-kit-core' {
    interface Register {
        dAppKit: typeof dAppKit;
    }
}
```

## Minimal (network-only, no explicit URLs)

Used by: tetris_game, sandbox_game, card_crawler

```typescript
import { createDAppKit } from '@mysten/dapp-kit-core';
import { SuiGrpcClient } from '@mysten/sui/grpc';

export const dAppKit = createDAppKit({
    networks: ['testnet', 'devnet'],
    defaultNetwork: 'testnet',
    enableBurnerWallet: import.meta.env.DEV,
    createClient(network) {
        return new SuiGrpcClient({ network });
    },
});

declare module '@mysten/dapp-kit-core' {
    interface Register {
        dAppKit: typeof dAppKit;
    }
}
```

> **Customization:** Add `'mainnet'` to `networks` and `GRPC_URLS` for production deployments.
