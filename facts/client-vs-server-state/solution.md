# Decoupling Client and Server State

Modern React architecture demands a strict boundary between the data you own (Client State) and the data you borrow (Server State). Treating them as the same entity leads to race conditions, manual cache invalidation bugs, and bloated global stores.

## The Mental Model

```text
┌─────────────────────────────────────────┐
│  Server Database                        │
│  (The absolute source of truth)         │
└────────────────┬────────────────────────┘
                 │
                 │ TanStack Query (Server State)
                 │ Handles: Fetching, Caching, Polling, Invalidation
                 ▼
┌─────────────────────────────────────────┐
│  Client Cache                           │
│  (A temporary, stale copy of API data)  │
└─────────────────────────────────────────┘

                 ▲
                 │
                 │ Zustand / Jotai (Client State)
                 │ Handles: Dark mode, Sidebar toggle, Draft inputs
                 │
┌─────────────────────────────────────────┐
│  Local UI State                         │
│  (Data that vanishes on page refresh)   │
└─────────────────────────────────────────┘
```

**The Rule of Thumb:** If the data survives a browser refresh (because it lives in a database), it is Server State. If it is lost on refresh, it is Client State.
