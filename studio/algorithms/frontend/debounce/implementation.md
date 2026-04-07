## Implementation in JavaScript

The core of a `debounce` function involves closures and `setTimeout`.

```javascript
function debounce(func, delay) {
  let timerId;

  return function (...args) {
    // If the function was called previously before the delay ended,
    // we clear the old timeout and start a new one.
    if (timerId) {
      clearTimeout(timerId);
    }

    timerId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}
```

### Usage
```javascript
const handleSearch = debounce((query) => {
  console.log("Searching for:", query);
}, 300);

// Only the last call will execute after 300ms of inactivity
handleSearch("A");
handleSearch("Ap");
handleSearch("App");
```
