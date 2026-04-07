```typescript
function quickSort(arr: number[], low: number = 0, high: number = arr.length - 1): number[] {
  if (low < high) {
    // pi is the partitioning index, arr[pi] is now at the right place
    const pi = partition(arr, low, high);

    // Recursively sort elements before partition and after partition
    quickSort(arr, low, pi - 1);
    quickSort(arr, pi + 1, high);
  }
  return arr;
}

function partition(arr: number[], low: number, high: number): number {
  // Choose the rightmost element as pivot
  const pivot = arr[high];
  
  // Index of the smaller element
  let i = low - 1;

  for (let j = low; j < high; j++) {
    // If the current element is smaller than or equal to pivot
    if (arr[j] <= pivot) {
      i++;
      // Swap arr[i] and arr[j]
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }

  // Swap arr[i+1] and arr[high] (or pivot)
  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  
  return i + 1;
}

// Example usage:
const unsortedArray = [10, 80, 30, 90, 40, 50, 70];
console.log("Unsorted Array:", unsortedArray);
console.log("Sorted Array:", quickSort([...unsortedArray]));
```
