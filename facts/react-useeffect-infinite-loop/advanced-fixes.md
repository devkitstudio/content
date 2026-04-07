## Fix 3: useCallback for Function Dependencies

When an effect depends on a function, that function gets recreated every render → new reference → effect re-fires.

```tsx
// BUG: fetchData is recreated every render → infinite loop
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  const fetchData = async () => {
    const res = await fetch(`/api/users/${userId}`);
    setUser(await res.json());
  };

  useEffect(() => {
    fetchData();
  }, [fetchData]); // ← new function ref every render = infinite loop
}

// FIX: wrap with useCallback
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  const fetchData = useCallback(async () => {
    const res = await fetch(`/api/users/${userId}`);
    setUser(await res.json());
  }, [userId]); // only recreated when userId changes

  useEffect(() => {
    fetchData();
  }, [fetchData]); // ✓ stable reference
}
```

## Fix 4: Move Fetch Logic Into the Effect

The simplest fix — don't create a separate function at all:

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let cancelled = false;
    const controller = new AbortController();

    async function load() {
      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        if (!cancelled) setUser(data);
      } catch (e) {
        if (e instanceof DOMException && e.name === 'AbortError') return;
        throw e;
      }
    }

    load();

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [userId]); // clean deps, no function reference issues
}
```

## Fix 5: Replace useEffect with React Query / SWR

For data fetching, `useEffect` is often the wrong tool entirely. Libraries like TanStack Query handle caching, deduplication, retries, and stale-while-revalidate out of the box — no infinite loop risk.

```tsx
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 min cache
  });

  if (isLoading) return <Skeleton />;
  if (error) return <Error />;
  return <div>{user.name}</div>;
}
// Zero useEffect, zero infinite loop risk
```

### SWR Alternative

```tsx
import useSWR from 'swr';

function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading } = useSWR(
    `/api/users/${userId}`,
    (url) => fetch(url).then(r => r.json())
  );
  // Same result, no useEffect needed
}
```

## Fix 6: useRef for Values That Shouldn't Trigger Re-renders

When you need to track a value across renders but it should NOT cause re-renders or re-trigger effects:

```tsx
// BUG: updating count in effect triggers re-render → effect re-fires
function ClickTracker() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const handler = () => setCount(count + 1); // stale closure + re-trigger
    window.addEventListener('click', handler);
    return () => window.removeEventListener('click', handler);
  }, [count]); // count changes → effect re-runs → adds listener again
}

// FIX: useRef for mutable value that doesn't trigger re-render
function ClickTracker() {
  const countRef = useRef(0);
  const [display, setDisplay] = useState(0);

  useEffect(() => {
    const handler = () => {
      countRef.current++;
      setDisplay(countRef.current); // update display when needed
    };
    window.addEventListener('click', handler);
    return () => window.removeEventListener('click', handler);
  }, []); // ← empty deps, runs once, no infinite loop
}
```
