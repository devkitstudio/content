# Throttle Implementation

A standard functional implementation in TypeScript.

### Primary Implementation (Timestamp Approach)

Using `Date.now()` is the most straightforward way to implement throttles, as it avoids complex timer manipulations.

```typescript
function throttle<T extends (...args: any[]) => void>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let lastRan = 0; // Timestamp of the last execution

  return function(this: any, ...args: Parameters<T>) {
    const now = Date.now();

    // If the time elapsed since the last execution is greater than the limit
    if (now - lastRan >= limit) {
      func.apply(this, args); // Execute immediately
      lastRan = now;          // Update the execution timestamp
    }
  };
}
```

### Leading & Trailing Edge Implementation (Timer Approach)

Using `setTimeout` allows implementing both the *leading edge* and a *trailing edge* guarantee (if the user stopped firing before the timer finished, we execute one last time at the end to catch up).

```typescript
function throttleTrailing<T extends (...args: any[]) => void>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle: boolean;
  let lastFunc: ReturnType<typeof setTimeout>;
  let lastRan: number;

  return function(this: any, ...args: Parameters<T>) {
    const context = this;
    
    // Setup for leading execution
    if (!inThrottle) {
      func.apply(context, args);
      lastRan = Date.now();
      inThrottle = true;
    } else {
      // Setup for trailing execution
      clearTimeout(lastFunc);
      
      lastFunc = setTimeout(function() {
        if ((Date.now() - lastRan) >= limit) {
          func.apply(context, args);
          lastRan = Date.now();
        }
      }, limit - (Date.now() - lastRan));
    }
  };
}
```

### Using Throttle

```javascript
// Attach to a scroll event
const handleScroll = throttle(() => {
  console.log("User is scrolling...");
}, 200);

window.addEventListener("scroll", handleScroll);
```
