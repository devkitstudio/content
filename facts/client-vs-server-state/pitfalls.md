## The "Double Source of Truth" Anti-Pattern

The most common mistake when adopting TanStack Query alongside Zustand is attempting to mirror server data into the global client store.

### ❌ The Fatal Anti-Pattern

```typescript
// WRONG: Fetching data via API and dumping it into Zustand
const useStore = create((set) => ({
  users: [],
  fetchUsers: async () => {
    const data = await fetch("/api/users").then((r) => r.json());
    set({ users: data }); // Danger: You now own a dead copy of server data
  },
}));
```

**Why this fails in production:**

1. You lose automatic cache invalidation.
2. If the user navigates away and comes back, the data does not refetch (stale data).
3. If another component deletes a user, you must manually write logic to splice the Zustand array, recreating a database locally.

### ✅ The Fix

Never copy Server State into Client State. Let TanStack Query act as your single source of truth for remote data. Retrieve it via the `useQuery` hook wherever needed; TanStack Query will deduplicate the network requests automatically.
