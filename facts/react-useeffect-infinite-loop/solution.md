### 1. Add the Empty Dependency Array
If you only want to run the effect ONCE when the component mounts, you must explicitly provide an empty array `[]`.

```tsx
function GoodComponent() {
  const [data, setData] = useState([]);
  
  // ✅ Empty array tells React: "ONLY run on mount"
  useEffect(() => {
    fetchData().then(res => setData(res));
  }, []); 
}
```

### 2. Beware of Referential Equality (The Silent Killer)
Even with a dependency array, objects and arrays can cause loops because `{}` is not equal to `{}` in JavaScript memory (`Object.is`).

```tsx
function TrickyComponent() {
  const [data, setData] = useState({});
  // 💀 A completely new object is created in memory on every render!
  const query = { limit: 10 }; 

  // React sees `query` is a new memory address, so it fires the effect again!
  useEffect(() => {
    fetchData(query).then(res => setData(res));
  }, [query]); 
}
```

**How to fix referential loops:**
1. Move static objects OUTSIDE the component.
2. Use `useMemo` to cache the object.
3. Stringify the object or pass primitive values: `[query.limit]` instead of `[query]`.
