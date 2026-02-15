# Utility Functions

All utilities are imported from `@mysten/sui/utils`.

```typescript
import {
  MIST_PER_SUI,
  SUI_DECIMALS,
  formatAddress,
  formatDigest,
  isValidSuiAddress,
  isValidSuiObjectId,
  isValidTransactionDigest,
  normalizeStructTag,
  normalizeSuiAddress,
  parseStructTag,
} from '@mysten/sui/utils';
```

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MIST_PER_SUI` | `1_000_000_000n` | 1 SUI = 1 billion MIST |
| `SUI_DECIMALS` | `9` | Number of decimal places |
| `SUI_TYPE_ARG` | `'0x2::sui::SUI'` | SUI coin type argument |
| `SUI_CLOCK_OBJECT_ID` | `'0x6'` | Clock shared object |
| `SUI_SYSTEM_STATE_OBJECT_ID` | `'0x5'` | System state object |

## Formatters

### `formatAddress(address: string): string`

Truncates an address for display: `0x1234...abcd`

```typescript
formatAddress('0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef');
// → '0x1234...abcd'
```

### `formatDigest(digest: string): string`

Truncates a transaction digest for display.

### `normalizeSuiAddress(address: string): string`

Pads a short address to the full 64-character hex format:

```typescript
normalizeSuiAddress('0x2');
// → '0x0000000000000000000000000000000000000000000000000000000000000002'
```

### `normalizeStructTag(tag: string): string`

Normalizes a Move struct tag to its canonical form:

```typescript
normalizeStructTag('0x2::coin::Coin<0x2::sui::SUI>');
// Fully normalizes all addresses within the tag
```

### `parseStructTag(tag: string)`

Parses a struct tag string into components:

```typescript
const parsed = parseStructTag('0x2::coin::Coin<0x2::sui::SUI>');
// { address: '0x2', module: 'coin', name: 'Coin', typeParams: [...] }
```

## Validators

### `isValidSuiAddress(address: string): boolean`

```typescript
isValidSuiAddress('0x1234...'); // true or false
```

### `isValidSuiObjectId(objectId: string): boolean`

```typescript
isValidSuiObjectId('0x1234...'); // true or false
```

### `isValidTransactionDigest(digest: string): boolean`

```typescript
isValidTransactionDigest('ABC123...'); // true or false
```

## Encoding Helpers

From `@mysten/sui/utils`:

```typescript
import { fromHex, toHex, fromBase64, toBase64 } from '@mysten/sui/utils';

// Hex
const bytes = fromHex('deadbeef');  // Uint8Array
const hex = toHex(bytes);           // 'deadbeef'

// Base64
const b64bytes = fromBase64('AAAA');
const b64str = toBase64(bytes);
```

## SUI / MIST Conversion

```typescript
// SUI to MIST
function suiToMist(sui: number): bigint {
  return BigInt(Math.floor(sui * Number(MIST_PER_SUI)));
}

// MIST to SUI (for display)
function mistToSui(mist: bigint | string): string {
  const mistBigInt = typeof mist === 'string' ? BigInt(mist) : mist;
  return (Number(mistBigInt) / Number(MIST_PER_SUI)).toFixed(SUI_DECIMALS);
}

// Usage
suiToMist(1.5);              // 1_500_000_000n
mistToSui('1500000000');     // '1.500000000'
mistToSui(1_500_000_000n);  // '1.500000000'
```
