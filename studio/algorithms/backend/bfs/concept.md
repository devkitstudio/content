# Breadth-First Search (BFS)

Breadth-First Search is an algorithm used to traverse or search in tree or graph data structures. It begins at a root node and explores all the neighboring nodes at the present depth before moving on to the nodes at the next depth level.

## Conceptual Analogy

Imagine dropping a stone into a calm pond. The ripples move outward in perfect, concentric circles. The wave hits all objects 1 foot away before it reaches any objects 2 feet away.

This is exactly how BFS searches:
1. Examine the starting position.
2. Examine all adjacent positions (depth 1).
3. Examine all positions adjacent to depth 1 (depth 2).
4. And so on until the target is found.

## Common Use Cases

1. **Shortest Path Problems:** Finding the shortest path out of a maze or minimum number of hops in an unweighted graph network (e.g., flight overlays).
2. **Social Networks:** Finding immediate friends, then friends of friends ("degrees of separation" / "connection depth").
3. **Web Crawlers:** Search engine algorithms use BFS to crawl the web, fetching all links on the current page before going deeper.
4. **Broadcast Routing:** Broadcasting packets over an OSI network gracefully reaches all neighbors layer-by-layer.

## Time and Space Complexity

Given `V` vertices (nodes) and `E` edges (connections):

- **Time Complexity:** `O(V + E)` - We theoretically must visit every node and boundary in the worst-case scenario.
- **Space Complexity:** `O(V)` - We must store pointers to the nodes we have "discovered but not yet visited" in a Queue. For a perfectly wide tree, the bottom layer could hold `V/2` nodes, directly translating to `O(V)` auxiliary space.
