## Architectural Trade-offs & Decision Tree

Do not use a 40KB library to toggle a modal, and do not write manual `useEffect` fetches to manage database records.

| Feature             | Zustand (Client State)                                     | TanStack Query (Server State)                                 |
| :------------------ | :--------------------------------------------------------- | :------------------------------------------------------------ |
| **Data Source**     | Browser memory                                             | Remote API / Database                                         |
| **Persistence**     | Lost on refresh (unless explicitly synced to localStorage) | Survives refresh (persisted on server)                        |
| **Data Ownership**  | Single browser session only                                | Shared across millions of clients                             |
| **Synchronization** | Manual updates only                                        | Automatic background refetching & invalidation                |
| **Bundle Size**     | ~2KB (Ultra-lightweight)                                   | ~40KB (Heavy, but replaces thousands of lines of boilerplate) |
| **Best Used For**   | Modals, themes, multi-step form drafts                     | User lists, dashboards, paginated data                        |

### State Selection Decision Tree

```text
Does the data live on a remote server?
├─ YES → Use TanStack Query
│   └─ Do mutations affect other users?
│       ├─ YES → Call queryClient.invalidateQueries() on success
│       └─ NO → Optimistically update local cache
│
└─ NO → Use Zustand or standard React useState
    └─ Is this state needed globally across distant components?
        ├─ YES → Zustand (Global Store)
        └─ NO → useState (Component-scoped)
```
