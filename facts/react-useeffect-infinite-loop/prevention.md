## How to Never Get Infinite Loops Again

### ESLint Rule: exhaustive-deps

The `react-hooks/exhaustive-deps` rule catches most infinite loop risks at build time:

```json
// .eslintrc.json
{
  "rules": {
    "react-hooks/exhaustive-deps": "error"
  }
}
```

```tsx
// ESLint will WARN on this:
useEffect(() => {
  fetchData(options); // options not in deps
}, []); // ⚠️ React Hook useEffect has a missing dependency: 'options'

// ESLint will ERROR on this:
useEffect(() => {
  setCount(count + 1); // count in deps = infinite loop
}, [count]); // ⚠️ potentially causes infinite loop
```

### The "Does This Belong in useEffect?" Checklist

Before writing `useEffect`, ask yourself:

| Question | If YES |
|----------|--------|
| Am I fetching data? | Use React Query / SWR instead |
| Am I transforming data for display? | Use `useMemo` instead |
| Am I handling a user event? | Use an event handler instead |
| Am I syncing with an external system? | `useEffect` is correct |
| Am I subscribing to something? | `useEffect` with cleanup is correct |

### React 19: The "You Might Not Need an Effect" Rule

```tsx
// BAD: useEffect to derive state
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter(i => i.active));
}, [items]); // unnecessary re-render cycle

// GOOD: compute during render
const [items, setItems] = useState([]);
const filteredItems = useMemo(
  () => items.filter(i => i.active),
  [items]
);

// GOOD: even simpler if cheap to compute
const [items, setItems] = useState([]);
const filteredItems = items.filter(i => i.active); // runs every render, fine for small lists
```

### Testing: Catch Infinite Loops in Dev

```tsx
// React Strict Mode (default in Next.js) runs effects TWICE in dev
// This catches effects that don't clean up properly

// next.config.js
module.exports = {
  reactStrictMode: true, // keep this ON
};

// Custom dev-only hook to detect excessive re-renders
function useRenderCount(componentName: string) {
  const count = useRef(0);
  count.current++;

  if (process.env.NODE_ENV === 'development' && count.current > 50) {
    console.error(
      `[RenderLoop] ${componentName} rendered ${count.current} times — possible infinite loop!`
    );
  }
}
```
