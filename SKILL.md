---
name: sui-on-chain-game-frontend-builder
description: Build Vite + React + TypeScript frontends for on-chain Sui games — project setup, wallet connection, transaction building, data reading, game state rendering, real-time updates, and best practices.
---

# Sui On-Chain Game Frontend Builder

## 🚨 Mandatory First Step

**Before writing ANY Sui game frontend code, you MUST read these two files:**

1. **Rule Zero** (below) — understand the core principles
2. **[dos_and_donts.md](references/dos_and_donts.md)** — avoid legacy imports and common pitfalls

Do NOT skip this step. Failing to read these will result in using deprecated APIs.

## Rule Zero

You are building a **React UI** that reads on-chain game state and submits game actions as transactions.
The game logic lives **entirely on-chain** (Move contracts). The frontend is a **view + action layer**, never a game engine.

You are **composing** the Sui TypeScript SDK, not reimplementing blockchain primitives.
Always use `@mysten/dapp-kit-react` for React hooks/components, `@mysten/dapp-kit-core` for `createDAppKit`, `@mysten/sui` for transactions/clients.
**Never** reference the legacy `@mysten/dapp-kit` or `SuiClient` from `@mysten/sui/client`.
`SuiJsonRpcClient` from `@mysten/sui/jsonRpc` **is valid** for data reads — see [client_api.md](references/client_api.md).

**Always:**
- Use **Vite + React + TypeScript** (no Next.js — games are client-only SPAs)
- Use `@mysten/dapp-kit-react` for wallet/client integration
- Use `@mysten/dapp-kit-core` for `createDAppKit`
- Use `SuiGrpcClient` for wallet/tx (via dApp Kit), `SuiJsonRpcClient` for standalone data reads
- Match frontend state models to on-chain Move structs
- Use polling or subscriptions to keep state fresh

**Never:**
- Put game logic in the frontend (the contract is the source of truth)
- Use SSR/SSG frameworks — games need client-side rendering
- Skip `waitForTransaction` before refreshing state post-action

## Packages

| Package | Import From | Purpose |
|---------|------------|---------|
| `@mysten/sui` | `@mysten/sui/transactions`, `@mysten/sui/grpc`, `@mysten/sui/jsonRpc`, etc. | Core SDK — transactions, clients, BCS, utils |
| `@mysten/dapp-kit-react` | `@mysten/dapp-kit-react` | React hooks, components, provider |
| `@mysten/dapp-kit-core` | `@mysten/dapp-kit-core` | Framework-agnostic core (`createDAppKit`) |
| `@tanstack/react-query` | `@tanstack/react-query` | Data fetching, caching, polling |
| `zustand` | `zustand` | Client-side UI state (selected entity, modals) |

```bash
npm create vite@latest my-game -- --template react-ts
cd my-game
npm i @mysten/dapp-kit-react @mysten/sui @tanstack/react-query zustand
```

## Decision Matrix — What Should I Read?

| I need to… | Read this reference |
|------------|-------------------|
| Scaffold a new Vite + React game project | [setup.md](references/setup.md) |
| Organize files and folders | [project_structure.md](references/project_structure.md) |
| Use React hooks for wallet/network/account | [hooks_api.md](references/hooks_api.md) |
| Build and execute transactions (PTB) | [transaction_patterns.md](references/transaction_patterns.md) |
| Query objects, balances, dynamic fields | [reading_data.md](references/reading_data.md) |
| Add wallet connect UI, manage connection | [wallet_patterns.md](references/wallet_patterns.md) |
| Configure SuiGrpcClient, SuiJsonRpcClient, or GraphQL | [client_api.md](references/client_api.md) |
| Keep game state fresh (polling, events, post-tx refresh) | [realtime_patterns.md](references/realtime_patterns.md) |
| Build game UI components & optimize perf | [ui_patterns.md](references/ui_patterns.md) |
| Format addresses, validate IDs, convert units | [utils.md](references/utils.md) |
| Copy a working recipe for a common task | [common_recipes.md](references/common_recipes.md) |
| Avoid common mistakes | [dos_and_donts.md](references/dos_and_donts.md) |

## Quick-Start Checklist

```
1. npm create vite@latest my-game -- --template react-ts
2. npm i @mysten/dapp-kit-react @mysten/sui @tanstack/react-query zustand
3. Create src/dApp-kit.ts  → createDAppKit({ networks, createClient })
4. Create src/lib/suiClient.ts → SuiJsonRpcClient for data reads
5. Add `declare module` for type registration
6. Create src/constants.ts → PACKAGE_ID, object IDs, game state enums
7. Wrap app in <DAppKitProvider> + <QueryClientProvider>
8. Add <ConnectButton /> to header
9. Read game state with suiClient + React Query (useQuery + refetchInterval)
10. Execute game actions with useDAppKit().signAndExecuteTransaction()
11. Refresh state post-tx with waitForTransaction + refetchQueries
```

## Prerequisite Skills

- **on_chain_game_builder** — for Move contract patterns and ECS game engine architecture
