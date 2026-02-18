# Template: main.tsx

Drop-in entry point. Near-identical across all 8 games.

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { DAppKitProvider } from '@mysten/dapp-kit-react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { dAppKit } from './dApp-kit';
import App from './App';
import './index.css';

const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 2_000,
            refetchOnWindowFocus: false,
        },
    },
});

createRoot(document.getElementById('root')!).render(
    <StrictMode>
        <QueryClientProvider client={queryClient}>
            <DAppKitProvider dAppKit={dAppKit}>
                <App />
            </DAppKitProvider>
        </QueryClientProvider>
    </StrictMode>,
);
```

> **Note:** If using additional CSS files (e.g. `animations.css`), add the import after `./index.css`.
