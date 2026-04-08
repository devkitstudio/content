## Architectural Trade-offs & Decision Tree

Do not use a heavy data-fetching library to toggle a UI modal, and do not write manual `useEffect` fetches to manage remote database records.

| Feature             | Zustand (Client State)                             | TanStack Query (Server State)                                       |
| :------------------ | :------------------------------------------------- | :------------------------------------------------------------------ |
| **Data Source**     | Browser memory                                     | Remote API / Database                                               |
| **Persistence**     | Ephemeral (Lost on refresh unless manually synced) | Persistent (Survives refresh, anchored on server)                   |
| **Data Ownership**  | Single browser session only                        | Shared across concurrent clients and sessions                       |
| **Synchronization** | Manual updates only                                | Automatic background refetching & invalidation                      |
| **Footprint Ratio** | Ultra-lightweight (Minimal impact)                 | Heavy (But eliminates massive boilerplate and manual caching logic) |
| **Best Used For**   | Modals, themes, multi-step form drafts             | User lists, dashboards, paginated data                              |

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
