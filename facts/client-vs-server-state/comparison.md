| Feature | Zustand/Jotai (Client) | TanStack Query (Server) |
|---------|-------|------|
| **Data source** | Browser memory | Server API |
| **Persistence** | Lost on refresh | Survives refresh (cached) |
| **Sharing** | Single browser only | Shared across clients |
| **Sync** | Manual | Automatic + invalidation |
| **Stale data** | You decide | Configurable TTL |
| **Updates** | Immediate | On mutation + refetch |
| **Offline** | Works | Queued until online |
| **Bundle size** | ~2KB | ~40KB |
| **Learning curve** | Easy | Medium |
| **Use case** | UI state | API data |

## Code Examples Side-by-Side

**Toggling a modal (CLIENT STATE)**
```typescript
// ✓ Zustand (correct)
const useStore = create((set) => ({
  isOpen: false,
  toggle: () => set((s) => ({ isOpen: !s.isOpen }))
}));

// ✗ TanStack Query (overkill)
// Don't do this - no server connection needed
```

**Displaying user list (SERVER STATE)**
```typescript
// ✗ Zustand (wrong)
const useStore = create((set) => ({
  users: [],
  fetchUsers: async () => { /* manual fetch */ }
}));
// Problems: no cache invalidation, no refetch on focus, manual sync

// ✓ TanStack Query (correct)
const { data: users } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json())
});
// Automatic: caching, refetch, invalidation, window focus
```

## Architecture Pattern

```
Perfect Setup:
├─ TanStack Query
│  └─ Manages: API data, caching, sync, invalidation
│
├─ Zustand
│  └─ Manages: UI state, preferences, temporary data
│
└─ React Local State (useState)
   └─ Manages: Component-scoped UI (form input, hover)
```

**Don't use:**
- Zustand for API data (TanStack Query better)
- TanStack Query for UI preferences (overkill)
- useState for cross-component UI state (use Zustand)

## Decision Tree

```
Does data live on the server?
├─ YES → Use TanStack Query
│   └─ Do mutations affect other users?
│       ├─ YES → invalidate on success
│       └─ NO → just update local state
│
└─ NO → Use Zustand or useState
    └─ Needed across components?
        ├─ YES → Zustand
        └─ NO → useState
```
