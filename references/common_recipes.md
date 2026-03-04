# Common Recipes

Copy-paste patterns for frequent Sui frontend tasks. All assume you have:
- `DAppKitProvider` set up (see [setup.md](setup.md))
- Hooks imported from `@mysten/dapp-kit-react`

---

## 1. Send SUI to an Address

```tsx
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { MIST_PER_SUI } from '@mysten/sui/utils';

function SendSui() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function send() {
    const tx = new Transaction();
    const [coin] = tx.splitCoins(tx.gas, [0.5 * Number(MIST_PER_SUI)]); // 0.5 SUI
    tx.transferObjects([coin], '0xRecipientAddress');

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      console.error('Failed:', result.FailedTransaction);
      return;
    }

    await client.waitForTransaction({ digest: result.Transaction.digest });
    console.log('Sent! Digest:', result.Transaction.digest);
  }

  return <button onClick={send}>Send 0.5 SUI</button>;
}
```

---

## 2. Call a Move Function

```tsx
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';

function CallMoveFunction() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function callFunction() {
    const tx = new Transaction();

    tx.moveCall({
      package: '0xYourPackageId',
      module: 'your_module',
      function: 'your_function',
      typeArguments: [],
      arguments: [
        tx.object('0xSharedObjectId'),
        tx.pure.u64(42),
        tx.pure.string('hello'),
      ],
    });

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw new Error('Transaction failed');
    }

    await client.waitForTransaction({
      digest: result.Transaction.digest,
      include: { effects: true },
    });
  }

  return <button onClick={callFunction}>Call Function</button>;
}
```

---

## 3. Fetch an Object and Display Fields

```tsx
import { useCurrentClient } from '@mysten/dapp-kit-react';
import { useEffect, useState } from 'react';

function ObjectDisplay({ objectId }: { objectId: string }) {
  const client = useCurrentClient();
  const [fields, setFields] = useState<Record<string, any> | null>(null);

  useEffect(() => {
    client.core
      .getObject({ objectId, options: { showContent: true } })
      .then((res) => {
        if (res.data?.content?.dataType === 'moveObject') {
          setFields(res.data.content.fields as Record<string, any>);
        }
      });
  }, [client, objectId]);

  if (!fields) return <p>Loading...</p>;

  return (
    <dl>
      {Object.entries(fields).map(([key, value]) => (
        <div key={key}>
          <dt>{key}</dt>
          <dd>{JSON.stringify(value)}</dd>
        </div>
      ))}
    </dl>
  );
}
```

---

## 4. List All Objects Owned by Connected User

```tsx
import { useCurrentClient, useCurrentAccount } from '@mysten/dapp-kit-react';
import { useEffect, useState } from 'react';

function OwnedObjects() {
  const client = useCurrentClient();
  const account = useCurrentAccount();
  const [objects, setObjects] = useState<any[]>([]);

  useEffect(() => {
    if (!account) return;

    client.core
      .getOwnedObjects({
        owner: account.address,
        options: { showContent: true, showType: true },
      })
      .then((res) => setObjects(res.data || []));
  }, [client, account]);

  if (!account) return <p>Connect wallet first</p>;

  return (
    <ul>
      {objects.map((obj) => (
        <li key={obj.data?.objectId}>
          {obj.data?.type} — {obj.data?.objectId}
        </li>
      ))}
    </ul>
  );
}
```

---

## 5. Mint an NFT by Calling a Contract

```tsx
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';

function MintNFT() {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function mint() {
    const tx = new Transaction();

    tx.moveCall({
      package: '0xNFTPackageId',
      module: 'nft',
      function: 'mint',
      arguments: [
        tx.pure.string('My NFT'),
        tx.pure.string('A description'),
        tx.pure.string('https://example.com/image.png'),
      ],
    });

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw new Error('Mint failed');
    }

    const txResult = await client.waitForTransaction({
      digest: result.Transaction.digest,
      include: { effects: true },
    });

    if (txResult.Transaction) {
      const created = txResult.Transaction.effects?.changedObjects?.find(
        (obj: any) => obj.idOperation === 'Created',
      );
      console.log('Minted NFT ID:', created?.objectId);
    }
  }

  return <button onClick={mint}>Mint NFT</button>;
}
```

---

## 6. Transfer an Owned Object

```tsx
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';

function TransferObject({ objectId }: { objectId: string }) {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function transfer(recipient: string) {
    const tx = new Transaction();
    tx.transferObjects([tx.object(objectId)], recipient);

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw new Error('Transfer failed');
    }

    await client.waitForTransaction({ digest: result.Transaction.digest });
    console.log('Transferred!');
  }

  return (
    <button onClick={() => transfer('0xRecipientAddress')}>
      Transfer Object
    </button>
  );
}
```

---

## 7. Check and Display SUI Balance

```tsx
import { useCurrentClient, useCurrentAccount } from '@mysten/dapp-kit-react';
import { MIST_PER_SUI } from '@mysten/sui/utils';
import { useEffect, useState } from 'react';

function BalanceDisplay() {
  const client = useCurrentClient();
  const account = useCurrentAccount();
  const [balance, setBalance] = useState<string | null>(null);

  useEffect(() => {
    if (!account) return;

    client.core
      .getBalance({ owner: account.address })
      .then((res) => {
        const sui = Number(BigInt(res.totalBalance)) / Number(MIST_PER_SUI);
        setBalance(sui.toFixed(4));
      });
  }, [client, account]);

  if (!account) return <p>Connect wallet</p>;
  if (balance === null) return <p>Loading...</p>;

  return <p>{balance} SUI</p>;
}
```

---

## 8. Using Codegen-Generated Move Calls

If you've run `@mysten/codegen` to generate TypeScript bindings:

```tsx
import { Transaction } from '@mysten/sui/transactions';
import { useDAppKit, useCurrentClient } from '@mysten/dapp-kit-react';
import { increment } from './contracts/counter/counter';

function IncrementCounter({ counterId }: { counterId: string }) {
  const dAppKit = useDAppKit();
  const client = useCurrentClient();

  async function doIncrement() {
    const tx = new Transaction();

    // codegen functions return (tx: Transaction) => result
    tx.add(increment({ arguments: [counterId] }));

    const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });

    if (result.FailedTransaction) {
      throw new Error('Increment failed');
    }

    await client.waitForTransaction({ digest: result.Transaction.digest });
  }

  return <button onClick={doIncrement}>Increment</button>;
}
```

---

## 9. Read ECS Entity Components

ECS entities store components as dynamic fields in a `Bag`. Use `getDynamicFields` → `multiGetObjects` → type-match to read them.

> [!NOTE]
> Requires `SuiJsonRpcClient` — see [client_api.md](client_api.md#suijsonrpcclient) for setup.

```tsx
import { useQuery } from '@tanstack/react-query';
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

const suiClient = new SuiJsonRpcClient({
  network: 'testnet',
  url: getJsonRpcFullnodeUrl('testnet'),
});

interface EntityComponents {
  health: { current: number; max: number };
  energy: { current: number; max: number };
  gold: number;
}

function useEntityComponents(entityId: string | null) {
  return useQuery<EntityComponents>({
    queryKey: ['entity', entityId],
    queryFn: async () => {
      // 1. Discover dynamic fields
      const dynFields = await suiClient.getDynamicFields({ parentId: entityId! });

      // 2. Batch-fetch all component objects
      const objects = await suiClient.multiGetObjects({
        ids: dynFields.data.map(df => df.objectId),
        options: { showContent: true, showType: true },
      });

      // 3. Parse by type name
      let health = { current: 0, max: 0 };
      let energy = { current: 0, max: 0 };
      let gold = 0;

      for (const obj of objects) {
        if (obj.data?.content?.dataType !== 'moveObject') continue;
        const type = obj.data.content.type ?? '';
        const fields = obj.data.content.fields as Record<string, any>;
        const value = fields.value?.fields ?? fields.value ?? fields;

        if (type.includes('Health')) {
          health = { current: Number(value.current), max: Number(value.max) };
        } else if (type.includes('Energy')) {
          energy = { current: Number(value.current), max: Number(value.max) };
        } else if (type.includes('Gold')) {
          gold = Number(value.amount);
        }
      }

      return { health, energy, gold };
    },
    enabled: !!entityId,
    refetchInterval: 2_000,
  });
}

function PlayerStats({ entityId }: { entityId: string }) {
  const { data } = useEntityComponents(entityId);
  if (!data) return <div>Loading...</div>;

  return (
    <div>
      <span>❤️ {data.health.current}/{data.health.max}</span>
      <span>⚡ {data.energy.current}/{data.energy.max}</span>
      <span>💰 {data.gold}</span>
    </div>
  );
}
```
