

# Merge Sort

**Merge Sort** is an efficient, general-purpose, and comparison-based sorting algorithm. Most implementations produce a **stable sort**, meaning that the implementation preserves the input order of equal elements in the sorted output. It is a **divide-and-conquer** algorithm.

## How it Works
1. **Divide**: Divide the unsorted list into `n` sublists, each containing one element (a list of one element is considered sorted).
2. **Conquer**: Repeatedly merge sublists to produce new sorted sublists until there is only one sorted list remaining. This will be the sorted list.

### The Merge Process
The core of the algorithm is the `merge` function. When merging two sorted arrays `A` and `B` into a new array `C`:
- Use two pointers, one at the beginning of `A` and one at the beginning of `B`.
- Compare the elements pointed to, push the smaller one into `C`, and advance the pointer of the array that held the smaller element.
- Repeat until one array is empty, then push the remaining elements of the other array into `C`.

## Performance
- **Time Complexity:** $O(n \log n)$ in all cases (Best, Average, Worst). This makes it highly predictable and a great choice when guaranteed performance is required.
- **Space Complexity:** $O(n)$ in its standard implementation because it requires an auxiliary array of size $n$ to merge the sublists. This is its main disadvantage compared to in-place sorts like Quick Sort.
