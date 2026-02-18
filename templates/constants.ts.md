# Template: constants.ts

Game constants extracted from the deployed Move contract. **Every value must match the contract.**

```typescript
// ═══════════════════════════════════════════════
// DEPLOYED CONTRACT IDS — update after `sui client publish`
// ═══════════════════════════════════════════════

export const PACKAGE_ID = '0x__REPLACE_WITH_PUBLISHED_PACKAGE_ID__';
export const WORLD_ID   = '0x__REPLACE_WITH_WORLD_OBJECT_ID__';
// Add more shared object IDs as needed:
// export const CLOCK_ID = '0x6';

// ═══════════════════════════════════════════════
// GAME STATES (must match Move contract constants)
// ═══════════════════════════════════════════════

export const STATE_LOBBY    = 0;
export const STATE_ACTIVE   = 1;
export const STATE_FINISHED = 2;
// Add more states matching your contract's const values

// ═══════════════════════════════════════════════
// ERROR CODES (must match Move contract abort codes)
// ═══════════════════════════════════════════════

export const GAME_ERROR_MAP: Record<number, string> = {
    // Extract from contract: const E_*: u64 = NNN;
    100: 'Invalid game state',
    101: 'Not the player',
    // ... map every abort code to a human-readable message
};
```

## How to Extract Values

### PACKAGE_ID & WORLD_ID
After running `sui client publish`, the output shows:
- `packageId` → use as `PACKAGE_ID`
- Created shared objects → find `World` type for `WORLD_ID`

### Game States
Search the `.move` file for `const STATE_*` or `const GAME_STATE_*` patterns.

### Error Codes
Search for `const E*: u64` patterns. Every `assert!(..., E_CODE)` in the contract
should have a corresponding entry in `GAME_ERROR_MAP`.
