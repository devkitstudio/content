# Optimistic Updates with Rollback

## The Problem

```javascript
// SLOW: Update shows only after server responds
function TodoItem({ todo, onToggle }) {
  const [isLoading, setIsLoading] = useState(false);

  const handleToggle = async () => {
    setIsLoading(true);
    // User sees loading spinner for 100-500ms
    await fetch(`/api/todos/${todo.id}`, {
      method: 'PATCH',
      body: JSON.stringify({ completed: !todo.completed }),
    });
    setIsLoading(false);
  };

  return (
    <li>
      {todo.text}
      <input
        type="checkbox"
        checked={todo.completed}
        disabled={isLoading}
        onChange={handleToggle}
      />
    </li>
  );
}
```

## Solution: Update UI Immediately, Rollback if Error

```javascript
function TodoItem({ todo }) {
  const [optimisticCompleted, setOptimisticCompleted] = useState(
    todo.completed
  );
  const [isUpdating, setIsUpdating] = useState(false);
  const [error, setError] = useState(null);

  const handleToggle = async () => {
    // Store previous state for rollback
    const previousCompleted = optimisticCompleted;

    // Update UI immediately (optimistic)
    setOptimisticCompleted(!optimisticCompleted);
    setIsUpdating(true);
    setError(null);

    try {
      // Make API request
      const response = await fetch(`/api/todos/${todo.id}`, {
        method: 'PATCH',
        body: JSON.stringify({ completed: !previousCompleted }),
      });

      if (!response.ok) {
        throw new Error('Failed to update');
      }

      // API succeeded - state already updated, nothing to do
      setIsUpdating(false);
    } catch (err) {
      // API failed - rollback to previous state
      setOptimisticCompleted(previousCompleted);
      setError('Failed to update todo');
      setIsUpdating(false);
    }
  };

  return (
    <li>
      {todo.text}
      <input
        type="checkbox"
        checked={optimisticCompleted}
        disabled={isUpdating}
        onChange={handleToggle}
      />
      {error && <span className="error">{error}</span>}
    </li>
  );
}
```

## TanStack Query Optimistic Updates

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function TodoItem({ todo }) {
  const queryClient = useQueryClient();

  const { mutate: toggleTodo, isPending } = useMutation({
    mutationFn: (id) =>
      fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        body: JSON.stringify({ completed: !todo.completed }),
      }).then((r) => r.json()),

    // Update cache immediately (optimistic)
    onMutate: async (id) => {
      // Cancel any pending refetches to avoid conflicts
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot old data for rollback
      const previousTodos = queryClient.getQueryData(['todos']);

      // Update cache optimistically
      queryClient.setQueryData(['todos'], (old) =>
        old.map((t) =>
          t.id === id ? { ...t, completed: !t.completed } : t
        )
      );

      return { previousTodos }; // Context for error handler
    },

    // If API succeeds, data already in cache
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },

    // Rollback on error
    onError: (error, id, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
    },
  });

  return (
    <li>
      {todo.text}
      <input
        type="checkbox"
        checked={todo.completed}
        disabled={isPending}
        onChange={() => toggleTodo(todo.id)}
      />
    </li>
  );
}
```

## Multiple Optimistic Updates

```javascript
function TodoList({ todos }) {
  const queryClient = useQueryClient();

  const { mutate: updateTodo } = useMutation({
    mutationFn: (data) =>
      fetch(`/api/todos/${data.id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }).then((r) => r.json()),

    onMutate: async (newTodo) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);

      queryClient.setQueryData(['todos'], (old) =>
        old.map((t) =>
          t.id === newTodo.id ? { ...t, ...newTodo } : t
        )
      );

      return { previousTodos };
    },

    onError: (error, newTodo, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
      // Show error toast
      showErrorToast('Failed to update todo');
    },

    onSuccess: () => {
      // Optional: refresh from server to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          {todo.text}
          <button onClick={() => updateTodo({ id: todo.id, text: 'New text' })}>
            Edit
          </button>
        </li>
      ))}
    </ul>
  );
}
```

## Optimistic Delete with Undo

```javascript
function TodoItem({ todo, onDelete }) {
  const queryClient = useQueryClient();
  const [showUndoToast, setShowUndoToast] = useState(false);

  const { mutate: deleteTodo } = useMutation({
    mutationFn: (id) =>
      fetch(`/api/todos/${id}`, { method: 'DELETE' }),

    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);

      // Remove from cache immediately
      queryClient.setQueryData(['todos'], (old) =>
        old.filter((t) => t.id !== id)
      );

      // Show undo toast
      setShowUndoToast(true);
      const undoTimeout = setTimeout(() => setShowUndoToast(false), 5000);

      return { previousTodos, undoTimeout, id };
    },

    onError: (error, id, context) => {
      // Restore on error
      queryClient.setQueryData(['todos'], context.previousTodos);
      setShowUndoToast(false);
    },

    onSuccess: () => {
      setShowUndoToast(false);
    },
  });

  const handleUndo = (context) => {
    // Restore previous state
    queryClient.setQueryData(['todos'], context.previousTodos);
    clearTimeout(context.undoTimeout);
    setShowUndoToast(false);
  };

  return (
    <>
      <li>
        {todo.text}
        <button onClick={() => deleteTodo(todo.id)}>Delete</button>
      </li>
      {showUndoToast && (
        <Toast>
          Deleted {todo.text}
          <button onClick={handleUndo}>Undo</button>
        </Toast>
      )}
    </>
  );
}
```

## Best Practices

1. **Always show loading state** for visual feedback
2. **Store previous state** before making API call
3. **Rollback on error** to maintain data consistency
4. **Show error messages** when rollback happens
5. **Consider timeout** for optimistic updates
6. **Validate locally** before optimistic update
7. **Provide undo option** for destructive operations
