```typescript
class BloomFilter {
  private size: number;
  private memory: boolean[];
  private seeds: number[];

  constructor(size = 100, numHashes = 3) {
    this.size = size;
    this.memory = new Array(size).fill(false);
    // Generate prime seeds for fake hashing
    this.seeds = [5381, 33, 53]; 
  }

  // A very simple hashing function (djb2 variant) used for demonstration
  private hash(item: string, seed: number): number {
    let hash = seed;
    for (let i = 0; i < item.length; i++) {
        hash = (hash * 33) ^ item.charCodeAt(i);
    }
    return Math.abs(hash) % this.size;
  }

  public add(item: string): void {
    for (const seed of this.seeds) {
        const index = this.hash(item, seed);
        this.memory[index] = true;
    }
  }

  public check(item: string): boolean {
    for (const seed of this.seeds) {
        const index = this.hash(item, seed);
        if (!this.memory[index]) {
            return false; // Definitely not present
        }
    }
    return true; // Might be present
  }
}
```
