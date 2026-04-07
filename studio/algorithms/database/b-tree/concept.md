# B-Tree

A B-Tree (often known as a Balanced Tree, though the 'B' formally has no single definition, sometimes attributed to its inventor Bayer) is a self-balancing search tree. In contrast to binary search trees that have at most two children per node, B-Trees are designed to have many children (a "branching factor" or "order").

They are fundamentally optimized for systems that read and write large blocks of data, making them the universal standard for Databases and File Systems.

## Why Databases Use B-Trees

When data grows beyond the capacity of RAM, it must be stored on a mechanical Hard Drive or SSD. Storage devices operate by reading contiguous "Blocks" (or Pages), usually 4KB or 8KB in size. 

Reading a single byte from a disk takes just as long as reading the entire 4KB block it lives in. Therefore, performing a standard BST (Binary Search Tree) search on a disk would be incredibly slow because each pointer traversal (`node.left` or `node.right`) requires an expensive Disk I/O operation to fetch a tiny node.

### The B-Tree Advantage
A B-Tree solves this by packing arrays of keys and arrays of child pointers into single nodes so that **one entire node fits perfectly into one Disk Page (4KB)**. 

If a B-Tree node can hold 500 keys:
- The root holds 500 keys.
- The 2nd level holds `500 * 500 = 250,000` keys.
- The 3rd level holds `500 * 250,000 = 125,000,000` keys.
- The 4th level holds **62.5 Billion** keys.

This means finding any record among 62.5 billion rows in a database requires a maximum of **4 Disk I/O reads** (Depth of 4). Often, the top 2 levels are fully cached in RAM, meaning you only ever do 1 or 2 Disk jumps.

## Rules of a B-Tree (Order $m$)
1. Every node has at most $m$ children.
2. Every internal node (except root) has at least $\lceil m/2 \rceil$ children.
3. Every node containing $k$ children contains exactly $k-1$ keys.
4. All leaves appear on the exact same depth level (perfectly balanced).
5. Keys within a node are stored in sorted ascending order.
