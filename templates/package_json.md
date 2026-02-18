# Template: package.json

Dependencies proven across all 8 game frontends. These exact versions compile and run.

```json
{
    "name": "__GAME_SLUG__-frontend",
    "private": true,
    "version": "0.0.0",
    "type": "module",
    "scripts": {
        "dev": "vite",
        "build": "tsc -b && vite build",
        "preview": "vite preview"
    },
    "dependencies": {
        "@mysten/dapp-kit-react": "^1.0.2",
        "@mysten/sui": "^2.4.0",
        "@tanstack/react-query": "^5.90.21",
        "react": "^19.2.0",
        "react-dom": "^19.2.0",
        "zustand": "^5.0.11"
    },
    "devDependencies": {
        "@types/react": "^19.2.7",
        "@types/react-dom": "^19.2.3",
        "@vitejs/plugin-react": "^5.1.1",
        "typescript": "~5.9.3",
        "vite": "^7.3.1"
    }
}
```

## Optional Dependencies

Add as needed:

| Package | Version | When to add |
|---------|---------|-------------|
| `phaser` | `^3.90.0` | Canvas-based 2D games (virus_game) |
| `framer-motion` | `^11.x` | Smooth animations |
| `lucide-react` | `^0.x` | Icons |
| `@eslint/js` | `^9.39.1` | Linting (add with eslint config) |

> **Replace** `__GAME_SLUG__` with your game's slug (e.g. `virus-game`, `card-crawler`).
