# Template: vite.config.ts

Vite configuration for game frontends.

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    base: '/__GAME_SLUG__/',
    plugins: [react()],
    resolve: {
        alias: {
            '@': '/src',
        },
    },
    build: {
        target: 'esnext', // gRPC transport needs modern JS
    },
});
```

> **Replace** `__GAME_SLUG__` with the game's slug from `example.json` (e.g. `virus-game`, `sokoban`).

## Minimal Variant

Some simpler games skip `resolve.alias` and `build.target`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    base: '/__GAME_SLUG__/',
    plugins: [react()],
});
```

## Chunk Splitting (Optional)

For large games, add manual chunk splitting to improve load times:

```typescript
build: {
    rollupOptions: {
        output: {
            manualChunks(id) {
                if (id.includes('node_modules/react') || id.includes('node_modules/react-dom')) {
                    return 'vendor-react';
                }
                if (id.includes('node_modules/@mysten')) {
                    return 'vendor-sui';
                }
                if (id.includes('node_modules/@tanstack')) {
                    return 'vendor-query';
                }
            },
        },
    },
},
```
