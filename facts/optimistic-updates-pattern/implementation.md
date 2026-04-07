# Advanced Patterns & Race Conditions

## Handling Race Conditions

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useRef } from 'react';

function RaceConditionSafeTodoItem({ todo }) {
  const queryClient = useQueryClient();
  const updateTimeoutRef = useRef(null);

  const { mutate: updateTodo } = useMutation({
    mutationFn: (data) =>
      fetch(`/api/todos/${data.id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }).then((r) => r.json()),

    onMutate: async (newData) => {
      // Cancel in-flight requests
      await queryClient.cancelQueries({ queryKey: ['todo', todo.id] });

      // Snapshot for rollback
      const previousTodo = queryClient.getQueryData(['todo', todo.id]);

      // Update optimistically
      queryClient.setQueryData(['todo', todo.id], (old) => ({
        ...old,
        ...newData,
      }));

      // Clear any pending timeout to avoid race condition
      if (updateTimeoutRef.current) {
        clearTimeout(updateTimeoutRef.current);
      }

      return { previousTodo };
    },

    onError: (error, newData, context) => {
      // Rollback
      queryClient.setQueryData(['todo', todo.id], context.previousTodo);
    },

    onSettled: () => {
      // Revalidate to ensure consistency with server
      queryClient.invalidateQueries({ queryKey: ['todo', todo.id] });
    },
  });

  const handleChange = (field, value) => {
    // Debounce updates to avoid too many requests
    if (updateTimeoutRef.current) {
      clearTimeout(updateTimeoutRef.current);
    }

    updateTimeoutRef.current = setTimeout(() => {
      updateTodo({ id: todo.id, [field]: value });
    }, 500);
  };

  return (
    <div>
      <input
        value={todo.text}
        onChange={(e) => handleChange('text', e.target.value)}
      />
    </div>
  );
}
```

## Optimistic with Server-Driven Updates

```javascript
function SmartTodoItem({ todo }) {
  const queryClient = useQueryClient();
  const [optimisticState, setOptimisticState] = useState(null);

  const { mutate: updateTodo } = useMutation({
    mutationFn: (data) =>
      fetch(`/api/todos/${data.id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }).then((r) => r.json()),

    onMutate: async (newData) => {
      await queryClient.cancelQueries({ queryKey: ['todo', todo.id] });
      const previousTodo = queryClient.getQueryData(['todo', todo.id]);

      // Store optimistic state locally
      setOptimisticState(newData);

      return { previousTodo };
    },

    onSuccess: (serverData) => {
      // Use server response as source of truth
      // (server may have modified the data)
      queryClient.setQueryData(['todo', todo.id], serverData);
      setOptimisticState(null);
    },

    onError: (error, newData, context) => {
      queryClient.setQueryData(['todo', todo.id], context.previousTodo);
      setOptimisticState(null);
    },
  });

  // Display optimistic state if available, else server state
  const displayTodo = optimisticState
    ? { ...todo, ...optimisticState }
    : todo;

  return (
    <li className={optimisticState ? 'optimistic' : ''}>
      {displayTodo.text}
    </li>
  );
}
```

## Batch Optimistic Updates

```javascript
function TodoBatch({ todos }) {
  const queryClient = useQueryClient();

  const { mutate: updateTodos } = useMutation({
    mutationFn: (updates) =>
      fetch('/api/todos/batch', {
        method: 'PATCH',
        body: JSON.stringify(updates),
      }).then((r) => r.json()),

    onMutate: async (updates) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);

      // Update all affected todos
      queryClient.setQueryData(['todos'], (old) =>
        old.map((todo) => {
          const update = updates.find((u) => u.id === todo.id);
          return update ? { ...todo, ...update } : todo;
        })
      );

      return { previousTodos };
    },

    onError: (error, updates, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
    },
  });

  const handleSelectAll = () => {
    const updates = todos.map((t) => ({
      id: t.id,
      completed: true,
    }));
    updateTodos(updates);
  };

  return (
    <>
      <button onClick={handleSelectAll}>Mark All Complete</button>
    </>
  );
}
```

## Optimistic with Loading Indicators

```javascript
function TodoItemWithUI({ todo }) {
  const queryClient = useQueryClient();
  const [isPending, setIsPending] = useState(false);
  const [hasError, setHasError] = useState(false);

  const { mutate: toggleTodo } = useMutation({
    mutationFn: () =>
      new Promise((resolve) => {
        setTimeout(() => resolve(true), 2000);
      }),

    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);

      setIsPending(true);
      setHasError(false);

      queryClient.setQueryData(['todos'], (old) =>
        old.map((t) =>
          t.id === todo.id ? { ...t, completed: !t.completed } : t
        )
      );

      return { previousTodos };
    },

    onError: (error, variables, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
      setIsPending(false);
      setHasError(true);

      // Clear error after 3 seconds
      setTimeout(() => setHasError(false), 3000);
    },

    onSuccess: () => {
      setIsPending(false);
    },
  });

  return (
    <div className={`todo ${hasError ? 'error' : ''}`}>
      <input
        type="checkbox"
        checked={todo.completed}
        disabled={isPending}
        onChange={() => toggleTodo()}
      />
      {todo.text}
      {isPending && <span className="spinner" />}
      {hasError && <span className="error-icon">!</span>}
    </div>
  );
}
```

## Validation Before Optimistic Update

```javascript
function ValidatedTodoUpdate({ todo }) {
  const { mutate: updateTodo } = useMutation({
    mutationFn: (data) =>
      fetch(`/api/todos/${data.id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }).then((r) => r.json()),

    onMutate: async (newData) => {
      // Validate before optimistic update
      const validation = validateTodoUpdate(newData);
      if (!validation.isValid) {
        throw new Error(validation.error);
      }

      // Now safe to update optimistically
      queryClient.setQueryData(['todo', todo.id], (old) => ({
        ...old,
        ...newData,
      }));

      return {};
    },
  });

  const handleChange = (field, value) => {
    // Local validation
    if (field === 'text' && value.length === 0) {
      showError('Text cannot be empty');
      return;
    }

    // Safe to update
    updateTodo({ id: todo.id, [field]: value });
  };

  return (
    <input
      value={todo.text}
      onChange={(e) => handleChange('text', e.target.value)}
    />
  );
}

function validateTodoUpdate(data) {
  if (data.text && data.text.length === 0) {
    return { isValid: false, error: 'Text required' };
  }
  if (data.dueDate && new Date(data.dueDate) < new Date()) {
    return { isValid: false, error: 'Due date must be in future' };
  }
  return { isValid: true };
}
```
