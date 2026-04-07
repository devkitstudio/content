## The Core Difference

**Client State: "Who controls this data?"**
- You (frontend) own it
- Local UI state
- Doesn't exist on server
- Example: form input, collapsed menu

**Server State: "Where does this data live?"**
- Server owns it
- Frontend is just a cache
- Multiple clients may access it
- Example: user list, current user

## Client State (Zustand/Jotai)

```typescript
// UI state that belongs to client only
import { create } from 'zustand';

const useUIStore = create((set) => ({
  isSidebarOpen: true,
  isDarkMode: false,
  toggleSidebar: () => set((s) => ({ isSidebarOpen: !s.isSidebarOpen })),
  toggleDarkMode: () => set((s) => ({ isDarkMode: !s.isDarkMode }))
}));

// Usage
export function App() {
  const isSidebarOpen = useUIStore((s) => s.isSidebarOpen);
  const toggleSidebar = useUIStore((s) => s.toggleSidebar);

  return <button onClick={toggleSidebar}>Toggle</button>;
}
```

**Good for:**
- UI preferences (theme, sidebar, modal open)
- Form state (before submission)
- Temporary local data
- Animation states

## Server State (TanStack Query)

```typescript
// Data that lives on server, we cache locally
import { useQuery, useMutation } from '@tanstack/react-query';

export function UserList() {
  // Fetch from server + cache
  const { data: users, isLoading } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('/api/users');
      return res.json();
    }
  });

  // Mutation + automatic cache invalidation
  const { mutate: deleteUser } = useMutation({
    mutationFn: async (userId: string) => {
      await fetch(`/api/users/${userId}`, { method: 'DELETE' });
    },
    onSuccess: () => {
      // Refetch users list
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });

  if (isLoading) return <div>Loading...</div>;

  return (
    <ul>
      {users?.map((user) => (
        <li key={user.id}>
          {user.name}
          <button onClick={() => deleteUser(user.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

**Good for:**
- Data from server (users, posts, comments)
- Shared data (other users see it)
- Cache management (invalidation, background sync)
- Real-time updates (polling, WebSocket)

## Mental Model

```
┌─────────────────────────────────────────┐
│  Server Database                        │
│  (source of truth)                      │
└────────────────┬────────────────────────┘
                 │
                 │ TanStack Query
                 │ (fetches & caches)
                 ▼
┌─────────────────────────────────────────┐
│  Client Cache                           │
│  (stale copy of server data)            │
└─────────────────────────────────────────┘

                 ▲
                 │
                 │ Zustand
                 │ (UI state)
                 │
┌─────────────────────────────────────────┐
│  UI Component State                     │
│  (form input, modals, etc)              │
└─────────────────────────────────────────┘
```

## Real Example: Blog Post Editor

```typescript
// Server state: post data from API
const { data: post } = useQuery({
  queryKey: ['posts', postId],
  queryFn: () => fetch(`/api/posts/${postId}`).then(r => r.json())
});

// Client state: UI/form state
const useEditorStore = create((set) => ({
  title: post?.title || '',
  content: post?.content || '',
  isDirty: false,
  setTitle: (title) => set({ title, isDirty: true }),
  setContent: (content) => set({ content, isDirty: true })
}));

// Mutation: save to server
const { mutate: savePost } = useMutation({
  mutationFn: (data) => fetch(`/api/posts/${postId}`, {
    method: 'PUT',
    body: JSON.stringify(data)
  }),
  onSuccess: () => {
    // Refetch post from server
    queryClient.invalidateQueries({ queryKey: ['posts', postId] });
    // Reset dirty flag
    useEditorStore.setState({ isDirty: false });
  }
});
```

## When to Use What

| Scenario | Use |
|----------|-----|
| Theme toggle | Client (Zustand) |
| Modal open/close | Client (Zustand) |
| Form input | Client (Zustand) |
| User list from API | Server (TanStack Query) |
| Pagination | Server (TanStack Query) |
| Search results | Server (TanStack Query) |
| Authenticated user | Server (TanStack Query) |
| Selected items in list | Client or Server? |

**Selected items?** Depends:
- If selection is temporary → Client
- If selection affects server → Server
- If shared across sessions → Server

## Don't Mix Them Up

```typescript
// WRONG: Storing server data in Zustand
const useStore = create((set) => ({
  users: [],
  fetchUsers: async () => {
    const data = await fetch('/api/users').then(r => r.json());
    set({ users: data });
  }
}));

// RIGHT: Use TanStack Query for server data
const { data: users } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json())
});
```

Why? TanStack Query handles:
- Caching
- Invalidation
- Refetch on window focus
- Stale data management
- Race conditions
- Deduplication
