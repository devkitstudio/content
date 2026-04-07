```typescript
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) {
    return arr;
  }

  const mid = Math.floor(arr.length / 2);
  const left = arr.slice(0, mid);
  const right = arr.slice(mid);

  return merge(mergeSort(left), mergeSort(right));
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0;
  let j = 0;

  // Compare elements from both arrays and push the smaller one
  while (i < left.length && j < right.length) {
    if (left[i] < right[j]) {
      result.push(left[i]);
      i++;
    } else {
      result.push(right[j]);
      j++;
    }
  }

  // Push remaining elements of the arrays (if any)
  return result.concat(left.slice(i)).concat(right.slice(j));
}

// Example usage:
const unsortedArray = [38, 27, 43, 3, 9, 82, 10];
console.log("Unsorted Array:", unsortedArray);
console.log("Sorted Array:", mergeSort(unsortedArray));
```
