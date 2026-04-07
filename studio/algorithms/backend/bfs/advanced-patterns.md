# Bidirectional Breadth-First Search

Standard Breadth-First Search radiates outward from the Start node in all directions like a ripple of water. If the shortest path between the start and destination is at depth $d$, standard BFS will explore roughly $b^d$ nodes (where $b$ is the branching factor).

For very large networks (like the Six Degrees of Kevin Bacon problem), $b^d$ becomes mathematically unscalable.

## The Strategy

If we know both the **Start Node** and **Target Node**, we can dramatically optimize standard BFS by searching from *both sides simultaneously*.

1. Ripple outwards from the Start Node using Queue A.
2. Ripple outwards from the Target Node using Queue B.
3. Every time we process a layer, check if the two ripples intersect!

When they collide, we reconstruct the path backward from the collision point. 

### Why is this faster?
If the distance between Start and Target is $d$:
- Unidirectional BFS searches $b^d$ nodes.
- Bidirectional BFS searches $b^{d/2} + b^{d/2}$ nodes.

**Example Comparison:**\
If branching factor is 10 and path length is 6 hops:\
Standard BFS visits roughly $10^6$ = **1,000,000 nodes.**\
Bidirectional BFS visits roughly $10^3 + 10^3$ = **2,000 nodes.**

This is a monumental optimization.
