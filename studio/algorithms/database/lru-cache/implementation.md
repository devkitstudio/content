# LRU Cache Implementation

The optimal implementation in TypeScript connects a Hash Map to a custom Doubly Linked List. 

*Note: In modern JavaScript, the built-in `Map` object actually maintains insertion order, so you can achieve an LRU Cache without a LinkedList by deleting and re-setting the key. However, the implementation below demonstrates the canonical algorithm used across all lower-level languages like C++ and Java.*

### Canonical Doubly Linked List + Hash Map

```typescript
// Define the Node for our Doubly Linked List
class Node<K, V> {
  key: K;
  value: V;
  prev: Node<K, V> | null = null;
  next: Node<K, V> | null = null;

  constructor(key: K, value: V) {
    this.key = key;
    this.value = value;
  }
}

class LRUCache<K, V> {
  private capacity: number;
  private cache: Map<K, Node<K, V>>;
  
  // Dummy head and tail nodes to avoid edge-case null checks
  private head: Node<keyof any, any>;
  private tail: Node<keyof any, any>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.cache = new Map();
    
    this.head = new Node("HEAD" as any, null);
    this.tail = new Node("TAIL" as any, null);
    
    // Link head <-> tail
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  // Helper to add a node right after the HEAD (Most Recently Used position)
  private addNode(node: Node<K, V>) {
    node.prev = this.head;
    node.next = this.head.next;

    this.head.next!.prev = node;
    this.head.next = node;
  }

  // Helper to remove any node securely
  private removeNode(node: Node<K, V>) {
    const prev = node.prev!;
    const next = node.next!;

    prev.next = next;
    next.prev = prev;
  }

  // Helper to move a node to the front (used when a node is accessed)
  private moveToHead(node: Node<K, V>) {
    this.removeNode(node);
    this.addNode(node);
  }

  // Retrieve an item takes O(1)
  get(key: K): V | -1 {
    const node = this.cache.get(key);
    
    if (!node) {
      return -1; // Cache miss
    }

    // Cache hit! Push it to the front as the most recently used
    this.moveToHead(node);
    
    return node.value;
  }

  // Insert or Update an item takes O(1)
  put(key: K, value: V): void {
    const node = this.cache.get(key);

    if (node) {
      // Update existing item and move to front
      node.value = value;
      this.moveToHead(node);
    } else {
      // Create a brand new node
      const newNode = new Node(key, value);
      
      // Cache overflowing? Evict LRU items
      if (this.cache.size >= this.capacity) {
        // Tail's prev is the Least Recently Used item
        const tailNode = this.tail.prev!;
        
        this.removeNode(tailNode);
        this.cache.delete(tailNode.key);
      }

      // Finally, add it to Cache and Linked List
      this.cache.set(key, newNode);
      this.addNode(newNode);
    }
  }
}
```
