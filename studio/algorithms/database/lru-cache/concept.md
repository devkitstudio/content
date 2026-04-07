# Least Recently Used (LRU) Cache

An LRU (Least Recently Used) Cache is an eviction policy algorithm used to manage memory within a fixed-size cache. When the cache hits its maximum capacity and a new item must be inserted, the LRU policy ejects the item that has not been accessed for the longest period of time.

## Conceptual Analogy

Imagine your desk: 
When you work, you place files on your desk. As your desk fills up, you need to make room for new files. Which file do you throw back into the filing cabinet? 
The most logical choice is the file that has been sitting untouched at the bottom of the pile for the longest time, assuming you probably won't need it soon.

## Time Complexity Requirements

For a cache to be efficient, both fetching an item and inserting an item must happen instantly.
- **Get (Read):** `O(1)` constant time.
- **Put (Write):** `O(1)` constant time.

## Data Structure Design

Achieving `O(1)` for both read and writes while maintaining an "Access Order" is impossible with a standard Array or a standard Hash Map alone.

- An **Array** provides sorting, but shifting elements when the "least recently used" drops off takes `O(N)`.
- A **Hash Map** provides `O(1)` access, but cannot inherently track ordering.

### The Double-Data-Structure Solution
The solution is a hybrid structure: **A Hash Map combined with a Doubly Linked List.**

1. The **Hash Map** maps a Key to a Linked List Node, providing `O(1)` retrieval.
2. The **Doubly Linked List** maintains the chronologic order of access. 
   - When an item is read or inserted, its node is detached from its current position and moved to the "Head" of the list (Most Recently Used).
   - When the cache capacity is exceeded, the node at the "Tail" (Least Recently Used) is dropped, which takes exactly `O(1)` operations.
