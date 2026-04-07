The infinite loop happens due to React's core rendering lifecycle:

1. **Render**: The component function runs.
2. **Effect**: `useEffect` runs because it's the first render (or dependencies changed).
3. **State Change**: Inside `useEffect`, you call `setData(...)`.
4. **Re-render Triggered**: Changing state forces React to re-render the component.
5. **The Trap**: If your `useEffect` lacks a dependency array, or has an unstable dependency (like an object or array created locally), React thinks the dependencies changed.
6. **Repeat**: Loop back to Step 1. Your browser crashes.

```tsx
function BadComponent() {
  const [data, setData] = useState([]);
  
  // ❌ Missing Dependency Array!
  // This runs after EVERY render. And sets state, triggering ANOTHER render.
  useEffect(() => {
    fetchData().then(res => setData(res));
  }); 

  return <div>{data.length}</div>;
}
```
