# Bubble Sort Implementation

A standard implementation in TypeScript:

```typescript
function bubbleSort(arr: number[]): number[] {
  let n = arr.length;
  let swapped: boolean;
  
  for (let i = 0; i < n - 1; i++) {
    swapped = false;
    for (let j = 0; j < n - i - 1; j++) {
      if (arr[j] > arr[j + 1]) {
        // Swap elements
        let temp = arr[j];
        arr[j] = arr[j + 1];
        arr[j + 1] = temp;
        swapped = true;
      }
    }
    // IF no two elements were swapped by inner loop, then break
    if (!swapped) break;
  }
  return arr;
}
```
