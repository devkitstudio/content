## 1. Uncleared Event Listeners & Subscriptions

The #1 cause. Components subscribe to global events or WebSockets but never unsubscribe on unmount.

```typescript
// LEAK: listener persists after component unmounts
useEffect(() => {
  window.addEventListener('resize', handleResize);
  const ws = new WebSocket('/live');
  ws.onmessage = handleMessage;
  // Missing cleanup!
}, []);

// FIX: always return a cleanup function
useEffect(() => {
  window.addEventListener('resize', handleResize);
  const ws = new WebSocket('/live');
  ws.onmessage = handleMessage;

  return () => {
    window.removeEventListener('resize', handleResize);
    ws.close();
  };
}, []);
```

## 2. Stale Closures Holding Large Data

A closure captures variables from its parent scope. If that closure lives forever (e.g., in a timer or event handler), the captured data can never be garbage-collected.

```typescript
// LEAK: setInterval closure holds entire `hugeDataset` forever
useEffect(() => {
  const hugeDataset = fetchAllProducts(); // 50MB array

  const id = setInterval(() => {
    console.log(hugeDataset.length); // closure captures hugeDataset
  }, 5000);

  // Even after unmount, if clearInterval is missed,
  // hugeDataset stays in memory
  return () => clearInterval(id);
}, []);
```

## 3. Detached DOM Nodes

Happens when you store references to DOM elements that have been removed from the document.

```typescript
// LEAK: ref to removed DOM node prevents GC
let cachedElement: HTMLElement | null = null;

function MyComponent() {
  const ref = useCallback((node: HTMLElement | null) => {
    cachedElement = node; // stored in module scope = never GC'd
  }, []);

  return <div ref={ref}>...</div>;
}

// FIX: clear references on unmount
useEffect(() => {
  return () => { cachedElement = null; };
}, []);
```

## 4. Forgotten AbortControllers

API calls that complete after component unmount still hold response data.

```typescript
// LEAK: response arrives after unmount, setState on unmounted component
useEffect(() => {
  fetch('/api/huge-dataset')
    .then(r => r.json())
    .then(data => setData(data));
}, []);

// FIX: abort on unmount
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/huge-dataset', { signal: controller.signal })
    .then(r => r.json())
    .then(data => setData(data))
    .catch(err => {
      if (err.name !== 'AbortError') throw err;
    });

  return () => controller.abort();
}, []);
```

## 5. Growing In-Memory Caches

State management stores (Redux, Zustand) that accumulate data across route navigations without ever clearing.

```typescript
// LEAK: every page visit adds more data, never removed
const useStore = create((set) => ({
  pageData: {} as Record<string, any>,
  loadPage: async (slug: string) => {
    const data = await fetchPage(slug);
    set(state => ({
      pageData: { ...state.pageData, [slug]: data } // grows forever
    }));
  },
}));

// FIX: implement LRU eviction or clear on navigation
const MAX_CACHED_PAGES = 10;
loadPage: async (slug) => {
  const data = await fetchPage(slug);
  set(state => {
    const entries = Object.entries({ ...state.pageData, [slug]: data });
    const trimmed = Object.fromEntries(entries.slice(-MAX_CACHED_PAGES));
    return { pageData: trimmed };
  });
},
```
