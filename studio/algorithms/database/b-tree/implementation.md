# B-Tree Search Implementation

B-Tree implementation for insertion and deletion is remarkably complex due to node-splitting (when a node exceeds $m$ elements, it splits and pushes a median key up to its parent) and node-merging (when a node drops below $\lceil m/2 \rceil$). 

However, searching a B-Tree is an elegant extension of Binary Search.

### Search Algorithm (In-Memory)

When searching a node, we apply a Binary Search across its sorted keys array to determine which child branch to descend into. 

```typescript
class BTreeNode {
  keys: number[];
  children: BTreeNode[];
  isLeaf: boolean;

  constructor(isLeaf: boolean) {
    this.keys = [];
    this.children = [];
    this.isLeaf = isLeaf;
  }
}

class BTree {
  root: BTreeNode;
  order: number; // Maximum children per node (m)

  constructor(order: number) {
    this.root = new BTreeNode(true);
    this.order = order;
  }

  // Returns the node containing the key, or null if not found
  search(key: number): BTreeNode | null {
    return this._search(this.root, key);
  }

  private _search(node: BTreeNode, key: number): BTreeNode | null {
    let i = 0;
    
    // Find the first key greater than or equal to k
    // Optimally, this linear scan is replaced by a local Binary Search 
    // within the node's keys array for large order sizes
    while (i < node.keys.length && key > node.keys[i]) {
      i++;
    }

    // Found the key
    if (i < node.keys.length && key === node.keys[i]) {
      return node; // Depending on DB implementation, this would return the disk offset pointer
    }

    // If we've hit a leaf and still haven't found it
    if (node.isLeaf) {
      return null;
    }

    // Descend into the appropriate child
    return this._search(node.children[i], key);
  }
}
```

### B+ Trees Extension
Modern databases (PostgreSQL, MySQL InnoDB) actually use a variant called **B+ Trees**. 
In a B+ tree, internal nodes ONLY store routing keys. Actual data records (or pointers to data) are strictly stored at the leaf nodes. Additionally, all leaf nodes form a Linked List horizontally, allowing incredibly fast scanning for range queries (`WHERE age BETWEEN 20 AND 30`).
