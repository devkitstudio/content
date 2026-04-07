## Iterative Implementation in JavaScript

```javascript
function binarySearch(arr, target) {
  let left = 0;
  let right = arr.length - 1;

  while (left <= right) {
    // Avoid integer overflow for very large arrays:
    const mid = left + Math.floor((right - left) / 2);

    if (arr[mid] === target) {
      return mid; // Target found
    }

    if (arr[mid] < target) {
      left = mid + 1; // It must be in the right half
    } else {
      right = mid - 1; // It must be in the left half
    }
  }

  return -1; // Target not found
}
```

### Constraints

The array **must be sorted** beforehand. Otherwise, the logic of "eliminating half" based on greater/lesser comparisons entirely fails.
