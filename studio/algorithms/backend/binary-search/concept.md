# Binary Search

Binary Search is an efficient algorithmic technique for finding a specific value in a **sorted array**. Its principle is elegantly simple: rather than checking every single element one by one, it repeatedly cuts the search space in half.

## Why use Binary Search?

- A linear search takes $O(N)$ time. For 1 million items, it might take 1 million checks.
- Binary Search runs in $O(\log N)$ time. For 1 million items, it takes at most **20 checks**!

## How it works

1. Check the middle element.
2. If it is the target, return.
3. If the target is smaller, repeat the search on the left half.
4. If larger, search the right half.
