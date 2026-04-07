# Diagnosing Re-renders with React DevTools & why-did-you-render

## React DevTools Profiler

### Step 1: Open Profiler Tab

1. Install React DevTools browser extension
2. Open DevTools → Profiler tab
3. Click record button (red circle)
4. Perform action (type in input)
5. Stop recording

### Step 2: Analyze Flame Chart

```
Timestamp: 0ms - Parent renders
  ├─ SearchInput: 0.2ms ✓ (memoized)
  ├─ Results: 2.1ms ✗ (re-rendered unnecessarily)
  └─ Sidebar: 0.3ms ✓ (memoized)
```

Red bars = expensive renders
Blue bars = fast renders

## Console.log Pattern

```javascript
function SearchInput({ value, onChange }) {
  console.count('SearchInput render');
  return <input value={value} onChange={onChange} />;
}

// Output:
// SearchInput render: 1
// SearchInput render: 2
// SearchInput render: 3 ← Too many!
```

## why-did-you-render Package

```bash
npm install --save-dev @welldone-software/why-did-you-render
```

### Setup

```javascript
// main.jsx or index.js
import whyDidYouRender from '@welldone-software/why-did-you-render';

whyDidYouRender(React, {
  trackAllPureComponents: true,
  trackHooks: {
    useCallback: true,
    useEffect: true,
    useMemo: true,
  },
});
```

### Mark Components to Track

```javascript
function SearchInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

SearchInput.whyDidYouRender = true;
```

### Console Output

```
SearchInput: rendered because:
  - "onChange" prop changed from [Function onChange] to [Function onChange]
    Old: ƒ onChange(e) { setInput(e.target.value); }
    New: ƒ onChange(e) { setInput(e.target.value); }
    - Which is different object identity

Tip: Wrap onChange with useCallback to fix!
```

## Profiling API

```javascript
import { unstable_trace as trace } from 'react';

trace('input-change', performance.now(), () => {
  setInput(value);
});

// Logs to DevTools Profiler with custom label
```

## Common Culprits

| Problem | Symptom | Solution |
|---------|---------|----------|
| Inline object props | `{ style: {} }` new every render | Extract to variable |
| Inline function props | `onClick={() => fn()}` | Use useCallback |
| Array props | `items={[].filter(...)}` | Use useMemo |
| No dependency array | `useEffect(() => {})` | Add dependencies |
| Expensive parent re-render | Child lists grow | Use React.memo + keys |

## Memo Debugging

```javascript
// Check if memo is working
const MemoComponent = React.memo(Component, (prevProps, nextProps) => {
  console.log('Comparing props:', prevProps, nextProps);
  return prevProps.value === nextProps.value; // true = skip render
});
```

## useCallback Debugging

```javascript
function Parent() {
  const handleClick = useCallback(() => {
    console.log('handleClick function created');
  }, [dependency]);

  return <Child onClick={handleClick} />;
}
```

## Performance Metrics

```javascript
// Measure render time
const start = performance.now();
// render something
const end = performance.now();
console.log(`Render took ${end - start}ms`);
```
