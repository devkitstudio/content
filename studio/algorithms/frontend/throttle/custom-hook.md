# React Hook: `useThrottle`

In React applications, using a raw `throttle` wrapper function can cause issues because React components re-render heavily, and standard scope closures reset on every render cycle.

To implement throttling safely inside a React Component, we must use `useRef` to track the state across rendering lifecycles without re-triggering new renders.

## Implementation

```tsx
import { useCallback, useRef } from "react";

export function useThrottle<Args extends any[]>(
  callback: (...args: Args) => void,
  limit: number
): (...args: Args) => void {
  
  // Track the timestamp of the last execution
  const lastRan = useRef(Date.now());
  const timeout = useRef<NodeJS.Timeout | null>(null);

  return useCallback(
    (...args: Args) => {
      const now = Date.now();

      // Check if we have surpassed the throttle limit organically
      if (now - lastRan.current >= limit) {
        callback(...args);
        lastRan.current = now;
      } else {
        // If not, clear the trailing timeout
        if (timeout.current) clearTimeout(timeout.current);
        
        // And set a new trailing edge execution
        timeout.current = setTimeout(() => {
          if (Date.now() - lastRan.current >= limit) {
            callback(...args);
            lastRan.current = Date.now();
          }
        }, limit - (now - lastRan.current));
      }
    },
    [callback, limit] // Only rebind if dependencies physically change
  );
}
```

### Usage Example

```tsx
function ScrollComponent() {
  const handleScroll = useThrottle(() => {
    console.log("Scrolled window!", window.scrollY);
  }, 100);

  useEffect(() => {
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, [handleScroll]);

  return <div style={{ height: "200vh" }}>Scroll down...</div>;
}
```
