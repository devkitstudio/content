## React Query + IndexedDB Pattern

```typescript
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import Dexie from 'dexie';

// Local database
const db = new Dexie('AppDB');
db.version(1).stores({
  todos: '++id, userId',
  syncQueue: '++id',
});

// React Query with offline persistence
export function useTodoList(userId: string) {
  const queryClient = useQueryClient();

  return useQuery({
    queryKey: ['todos', userId],
    queryFn: async () => {
      try {
        // Try server first
        const response = await fetch(`/api/todos?userId=${userId}`);
        const todos = await response.json();

        // Store locally for offline
        await db.todos.bulkPut(todos);
        return todos;
      } catch (e) {
        // Network error: use local cache
        return db.todos.where('userId').equals(userId).toArray();
      }
    },
  });
}

// Optimistic update with conflict handling
export function useUpdateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (todo: Todo) => {
      // Optimistic update
      queryClient.setQueryData(['todos'], (old: Todo[]) =>
        old.map(t => t.id === todo.id ? todo : t)
      );

      // Save locally
      await db.todos.put(todo);
      await db.syncQueue.add({
        type: 'UPDATE',
        data: todo,
        timestamp: Date.now(),
        synced: false,
      });

      // Try to sync
      try {
        const response = await fetch(`/api/todos/${todo.id}`, {
          method: 'PATCH',
          body: JSON.stringify(todo),
        });

        if (response.status === 409) {
          // Conflict: server has newer version
          const serverTodo = await response.json();
          const merged = merge(todo, serverTodo);
          await db.todos.put(merged);
          return merged;
        }

        const updated = await response.json();
        await db.syncQueue.where('id').equals(todo.id).delete();
        return updated;
      } catch (e) {
        // Offline: will retry on reconnect
        return todo;
      }
    },

    onError: (error) => {
      // Revert optimistic update on error
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// Background sync
export function useSyncOnline() {
  useEffect(() => {
    const syncPending = async () => {
      const pending = await db.syncQueue.where('synced').equals(false).toArray();

      for (const item of pending) {
        try {
          const response = await fetch(`/api/todos/${item.data.id}`, {
            method: 'PATCH',
            body: JSON.stringify(item.data),
          });

          if (response.ok) {
            await db.syncQueue.update(item.id, { synced: true });
          }
        } catch (e) {
          // Keep retrying
        }
      }
    };

    window.addEventListener('online', syncPending);
    return () => window.removeEventListener('online', syncPending);
  }, []);
}

// Helper: merge function
function merge(local: Todo, remote: Todo): Todo {
  return {
    ...local,
    // Server version is authoritative for timestamps
    updatedAt: remote.updatedAt > local.updatedAt ? remote.updatedAt : local.updatedAt,
    // Field-level merge
    ...Object.entries(remote).reduce(
      (acc, [key, val]) => {
        // If local changed after remote, keep local
        if (local[key as keyof Todo] !== val) {
          acc[key as keyof Todo] = val;
        }
        return acc;
      },
      {} as Partial<Todo>
    ),
  };
}
```

## Usage in Component

```tsx
function TodoApp() {
  const { data: todos, isPending } = useTodoList(userId);
  const updateTodo = useUpdateTodo();
  useSyncOnline();

  if (isPending) return <div>Loading...</div>;

  return (
    <div>
      {todos?.map(todo => (
        <div key={todo.id}>
          <input
            value={todo.title}
            onChange={(e) =>
              updateTodo.mutate({ ...todo, title: e.target.value })
            }
          />
        </div>
      ))}
    </div>
  );
}
```
