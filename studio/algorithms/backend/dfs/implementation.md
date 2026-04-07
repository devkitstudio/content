```typescript
interface Graph {
  [vertex: string]: string[];
}

function depthFirstSearch(graph: Graph, startNode: string) {
  const visited = new Set<string>();
  
  function explore(node: string) {
    if (visited.has(node)) return;
    
    // Process node
    console.log(`Visited: ${node}`);
    visited.add(node);
    
    // Visit all adjacent (neighbor) nodes recursively
    const neighbors = graph[node] || [];
    for (const neighbor of neighbors) {
      explore(neighbor);
    }
  }

  // Start the recursive exploration
  explore(startNode);
}

// Example usage:
const myGraph: Graph = {
  'A': ['B', 'C'],
  'B': ['D', 'E'],
  'C': ['F'],
  'D': [],
  'E': ['F'],
  'F': []
};

depthFirstSearch(myGraph, 'A');
// Output trajectory: 
// A -> B -> D -> E -> F -> C
```
