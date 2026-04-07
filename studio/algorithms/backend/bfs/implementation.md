# Breadth-First Search Implementation

BFS utilizes a **Queue** (First-In, First-Out) data structure to keep track of nodes to visit.

### Graph Traversal Implementation
To traverse a Graph, we must maintain a `visited` set to avoid infinitely looping in cyclic graphs.

```typescript
function bfsGraphSearch(graph: Map<string, string[]>, startNode: string, targetNode: string): boolean {
  // Edge Case: Starting at the target
  if (startNode === targetNode) return true;

  const queue: string[] = [startNode];
  const visited = new Set<string>();
  
  visited.add(startNode);

  while (queue.length > 0) {
    const current = queue.shift()!;
    
    // Found our target!
    if (current === targetNode) {
      return true;
    }

    // Get all neighbors
    const neighbors = graph.get(current) || [];
    
    // Evaluate neighbors
    for (const neighbor of neighbors) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);  // Mark visited immediately to prevent duplicate enqueues
        queue.push(neighbor);
      }
    }
  }

  // Exhausted the graph, target not found
  return false;
}
```

### Tree Traversal Implementation
In an acyclic directed tree, a `visited` set is not strictly required because there is no way to travel backward upwards.

```typescript
interface TreeNode {
  value: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function bfsTreeTraversal(root: TreeNode | null): number[] {
  if (!root) return [];
  
  const result: number[] = [];
  const queue: TreeNode[] = [root];
  
  while (queue.length > 0) {
    const node = queue.shift()!;
    
    result.push(node.value);
    
    if (node.left) queue.push(node.left);
    if (node.right) queue.push(node.right);
  }
  
  return result;
}
```
