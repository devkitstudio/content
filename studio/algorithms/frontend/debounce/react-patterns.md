# React Patterns for Debounce

When implementing a `.debounce` function inside a React app (e.g. for a Search Bar Input), developers face a strict lifecycle boundary issue because React functions re-render fundamentally, destroying variable contexts.

## The Anti-Pattern

```tsx
export function SearchBar() {
  const [query, setQuery] = useState("");

  // INCORRECT ❌
  // Every time `query` changes, React re-executes this ENTIRE function.
  // The old `debounce` scope is thrown away, and a brand new one is created.
  // The execution limit will NEVER be respected!
  const fetchResults = debounce((q: string) => {
    Api.search(q);
  }, 300);

  return <input onChange={e => {
    setQuery(e.target.value);
    fetchResults(e.target.value);
  }} />
}
```

## The Correct `useMemo` Pattern

To preserve the singular Debounce Timeout boundary across multiple render cycles, it must be persisted:

```tsx
export function SearchBar() {
  const [query, setQuery] = useState("");

  // CORRECT ✅
  // The exact same debounced function reference survives across re-renders
  const fetchResults = useMemo(() => 
    debounce((q: string) => {
      Api.search(q);
    }, 300)
  , []); // Never rebuild the debounce reference

  // ...
```

## Considerations regarding `useCallback`
Developers commonly confuse `useCallback` and `useMemo` for this task.
While `useCallback(() => debounce(...), [])` returns the *function that calculates the debounce wrapper*, it doesn't execute and store the result recursively. Always use `useMemo` when invoking structural Wrappers!
