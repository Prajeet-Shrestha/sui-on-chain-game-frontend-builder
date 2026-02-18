# Template: stores/uiStore.ts

Zustand store for client-only UI state. Core shape is consistent across all games.

```typescript
import { create } from 'zustand';

interface UIStore {
    // Active game session object ID (null = no game in progress)
    sessionId: string | null;
    setSessionId: (id: string | null) => void;

    // Transaction pending state
    isPending: boolean;
    setIsPending: (pending: boolean) => void;

    // User-facing error message
    error: string | null;
    setError: (error: string | null) => void;
}

export const useUIStore = create<UIStore>((set) => ({
    sessionId: null,
    setSessionId: (id) => set({ sessionId: id }),
    isPending: false,
    setIsPending: (pending) => set({ isPending: pending }),
    error: null,
    setError: (error) => set({ error }),
}));
```

## Common Extensions

Add fields as needed for your game:

```typescript
// Level select screen
selectedLevel: number | null;
setSelectedLevel: (level: number | null) => void;

// Modal visibility
showModal: boolean;
toggleModal: () => void;

// Selected entity (for tactics / grid games)
selectedEntity: string | null;
selectEntity: (id: string | null) => void;
```
