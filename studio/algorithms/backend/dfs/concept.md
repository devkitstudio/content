

# Depth-First Search (DFS)

**Depth-First Search (DFS)** is an algorithm used to traverse or search through a graph or tree data structure. The strategy is to go as deep as possible down a single path until you hit a dead end, and then backtrack to explore other branches.

## How it Works
1. **Start** at a chosen root node.
2. Mark the node as **visited** so you don't process it again (important for graphs with cycles).
3. Look at an adjacent unvisited node.
4. **Recursively** call DFS on that adjacent node (diving deeper).
5. If a node has no unvisited adjacent nodes, **backtrack** to the previous node and continue exploring.

## Use Cases
- Topological Sorting (e.g., resolving dependency trees like npm or apt).
- Finding Connected Components.
- Solving mazes and puzzles (like Sudoku).
- Cycle detection in graphs.

## BFS vs DFS
While **BFS** explores layer-by-layer (like ripples in a pond) and is great for finding the shortest path, **DFS** is more like exploring a maze by keeping your hand on the wall. DFS uses a **Stack** (either explicitly or via the system call stack with recursion), whereas BFS uses a **Queue**.
