---
name: sui-on-chain-game-frontend-builder
description: Build Vite + React + TypeScript frontends for on-chain Sui games — scaffolding, contract integration, wallet connection, transaction building, game state rendering, and best practices. Battle-tested across 8 game frontends.
---

# Sui On-Chain Game Frontend Builder

## 🚨 Mandatory First Steps

**Before writing ANY Sui game frontend code, you MUST read in order:**

1. **Rule Zero** (below)
2. **[dos_and_donts.md](references/dos_and_donts.md)** — avoid legacy imports and common pitfalls
3. **[scaffolding_protocol.md](references/scaffolding_protocol.md)** — the step-by-step process you MUST follow
4. **[contract_integration.md](references/contract_integration.md)** — how to read the Move contract and generate types/parsers/hooks

Do NOT skip these. Failing to follow the protocol will produce broken frontends.

## Rule Zero

You are building a **React UI** that reads on-chain game state and submits game actions as transactions.
The game logic lives **entirely on-chain** (Move contracts). The frontend is a **view + action layer**, never a game engine.

You are **composing** the Sui TypeScript SDK, not reimplementing blockchain primitives.
Always use `@mysten/dapp-kit-react` for React hooks/components, `@mysten/dapp-kit-core` for `createDAppKit`, `@mysten/sui` for transactions/clients.
**Never** reference the legacy `@mysten/dapp-kit` or `SuiClient` from `@mysten/sui/client`.
`SuiJsonRpcClient` from `@mysten/sui/jsonRpc` **is valid** for data reads — see [client_api.md](references/client_api.md).

**Always:**
- Use **Vite + React + TypeScript** (no Next.js — games are client-only SPAs)
- Follow the **Scaffolding Protocol** — do not invent the file creation order
- Read the **Move contract first** before writing any TypeScript
- Use templates from `templates/` for boilerplate files
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

## Scaffolding Protocol (Summary)

Follow [scaffolding_protocol.md](references/scaffolding_protocol.md) for full details. The phases are:

```
Phase 1: Read the Move contract → extract structs, functions, constants, shared objects
Phase 2: Scaffold boilerplate → copy templates (main.tsx, dApp-kit.ts, suiClient.ts, etc.)
Phase 3: Generate integration layer → constants.ts, types.ts, parsers.ts
Phase 4: Generate hooks → useGameActions.ts, useGameSession.ts
Phase 5: Build game components → App.tsx, GameBoard.tsx, Header.tsx, GameOver.tsx, index.css
Phase 6: Install & verify → npm install && npm run dev
```

## Contract → Frontend Pipeline

See [contract_integration.md](references/contract_integration.md) for the full workflow:

```
Move struct   → lib/types.ts      (TypeScript interfaces)
Move consts   → constants.ts      (states, error codes, IDs)
Move fields   → lib/parsers.ts    (JSON-RPC → typed objects)
Move pub funs → hooks/useGameActions.ts  (PTB builders)
```

## Templates (Copy-Paste Boilerplate)

Drop-in files extracted from 8 working game frontends:

| Template | File it creates |
|----------|----------------|
| [main.tsx.md](templates/main.tsx.md) | `src/main.tsx` |
| [dapp_kit.ts.md](templates/dapp_kit.ts.md) | `src/dApp-kit.ts` |
| [sui_client.ts.md](templates/sui_client.ts.md) | `src/lib/suiClient.ts` |
| [ui_store.ts.md](templates/ui_store.ts.md) | `src/stores/uiStore.ts` |
| [constants.ts.md](templates/constants.ts.md) | `src/constants.ts` (skeleton) |
| [use_game_actions.ts.md](templates/use_game_actions.ts.md) | `src/hooks/useGameActions.ts` |
| [use_game_session.ts.md](templates/use_game_session.ts.md) | `src/hooks/useGameSession.ts` |
| [vite_config.ts.md](templates/vite_config.ts.md) | `vite.config.ts` |
| [index_html.md](templates/index_html.md) | `index.html` |
| [package_json.md](templates/package_json.md) | `package.json` |

## Decision Matrix — What Should I Read?

| I need to… | Read this reference |
|------------|-------------------|
| Create a new game frontend from scratch | [scaffolding_protocol.md](references/scaffolding_protocol.md) |
| Read a Move contract and generate TS types/hooks | [contract_integration.md](references/contract_integration.md) |
| Find patterns from existing games | [game_examples.md](references/game_examples.md) |
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
| Set up project from scratch | [setup.md](references/setup.md) |

## Real Game Examples

8 battle-tested frontends in `examples/`. See [game_examples.md](references/game_examples.md) for the full catalog.

| Game | Best for learning |
|------|------------------|
| **virus_game** | Multi-step PTB, Phaser canvas, local simulation |
| **sokoban** | Level select, directional input, local preview |
| **card_crawler** | Card UI, animations, complex components |
| **tactics_ogre** | Multi-entity, turn-based, many hooks |
| **tetris_game** | Real-time tick, keyboard input |
| **maze_game** | Pathfinding, fog of war |
| **flappy_bird** | Physics loop, collision |
| **sandbox_game** | Entity editor, creative mode |

## Prerequisite Skills

- **on_chain_game_builder** — for Move contract patterns and ECS game engine architecture
