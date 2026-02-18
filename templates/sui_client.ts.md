# Template: lib/suiClient.ts

Standalone JSON-RPC client for data reads. 100% identical across all 8 games.

```typescript
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

export const suiClient = new SuiJsonRpcClient({
    network: 'testnet',
    url: getJsonRpcFullnodeUrl('testnet'),
});
```

> **Why separate from dApp Kit?** The `SuiGrpcClient` in dApp Kit is tied to the wallet connection. This `SuiJsonRpcClient` can read data before any wallet is connected — useful for displaying game state on load.
