# B+ Tree: The Industry Standard

While classical B-Trees perfectly solve the depth constraint Problem of Disk I/O storage devices, almost all modern databases (e.g. MySQL, PostgreSQL) utilize a variation termed a **B+ Tree**.

## What is a B+ Tree?

In a standard B-Tree, every node (Leaf AND Internal node) can store raw Data rows. This allows early-exit scenarios where if you get lucky, you can find your database record at Depth 1 or 2.

In a **B+ Tree**, the architecture forces two major systematic changes:

1. **Only leaves contain real data payloads.** Internal nodes strictly contain keys (used only as routing signposts).
2. **Leaf nodes form a horizontal doubly-linked list.** Every leaf points to the next adjacent leaf sequentially.

## Why is it superior to classical B-Trees?

1. **Massive Capacity at Top Layers**  
   Because internal nodes do not hold heavy data rows, they can store thousands of raw keys. This astronomically increases the Branching Factor $m$. A B+ Tree can hold billions of keys while remaining at a depth of 3!

2. **O(1) Range Queries**  
   SQL `SELECT * FROM users WHERE age BETWEEN 20 AND 30` executes incredibly fast. The database simply traverses down to find `20` (costing $O(\log N)$), and then just iterates the linked-list leaves forward until it hits `30`! In a standard B-Tree, it would bounce constantly up and down the tree branches to pick up nodes spanning ranges.
