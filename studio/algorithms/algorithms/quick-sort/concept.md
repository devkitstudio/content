

# Quick Sort

**Quick Sort** is a highly efficient sorting algorithm and is based on a **divide-and-conquer** strategy. Developed by British computer scientist Tony Hoare in 1959, it is still one of the most commonly used algorithms for sorting.

## How it Works
The algorithm works by selecting a "pivot" element from the array and partitioning the other elements into two sub-arrays, according to whether they are less than or greater than the pivot.

1. **Pick a Pivot**: Select an element from the array. This can be the first, last, middle, or a randomly chosen element.
2. **Partitioning**: Rearrange the array so that all elements with values less than the pivot come before it, and all elements with values greater than the pivot come after it. After this partitioning, the pivot is in its final position.
3. **Recursion**: Recursively apply the above steps to the sub-array of elements with smaller values and the sub-array of elements with greater values.

## Performance
- **Time Complexity:**
  - *Best & Average Case*: $O(n \log n)$ – This happens when the pivot consistently divides the array into two nearly equal halves.
  - *Worst Case*: $O(n^2)$ – This occurs when the array is already sorted (or reverse sorted) and the algorithm consistently picks the first or last element as the pivot.
- **Space Complexity:** $O(\log n)$ due to the call stack size.

Quick Sort is generally faster in practice than other $O(n \log n)$ algorithms like Merge Sort because its inner loop can be efficiently implemented on most architectures.
