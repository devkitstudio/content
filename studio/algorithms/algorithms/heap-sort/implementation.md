# Heap Sort Implementation

A standard implementation in TypeScript:

```typescript
function heapify(arr: number[], N: number, i: number) {
  let largest = i; // Initialize largest as root
  let l = 2 * i + 1; // left = 2*i + 1
  let r = 2 * i + 2; // right = 2*i + 2

  // If left child is larger than root
  if (l < N && arr[l] > arr[largest])
    largest = l;

  // If right child is larger than largest so far
  if (r < N && arr[r] > arr[largest])
    largest = r;

  // If largest is not root
  if (largest != i) {
    let swap = arr[i];
    arr[i] = arr[largest];
    arr[largest] = swap;

    // Recursively heapify the affected sub-tree
    heapify(arr, N, largest);
  }
}

function heapSort(arr: number[]): number[] {
  let N = arr.length;

  // Build heap (rearrange array)
  for (let i = Math.floor(N / 2) - 1; i >= 0; i--)
    heapify(arr, N, i);

  // One by one extract an element from heap
  for (let i = N - 1; i > 0; i--) {
    // Move current root to end
    let temp = arr[0];
    arr[0] = arr[i];
    arr[i] = temp;

    // call max heapify on the reduced heap
    heapify(arr, i, 0);
  }
  
  return arr;
}
```
