```typescript
type TaskFn<T> = () => Promise<T>;

/**
 * Executes a list of tasks with a concurrency limit.
 * 
 * @param tasks An array of functions that return a Promise.
 * @param concurrency The maximum number of promises to run at the same time.
 * @returns Array of resolved results in the same order as tasks.
 */
async function promisePool<T>(tasks: TaskFn<T>[], concurrency: number): Promise<T[]> {
  const results: T[] = [];
  const executing = new Set<Promise<void>>();

  for (const [index, task] of tasks.entries()) {
    // Wrap the task to store its result and automatically remove itself from the executing set
    const p = Promise.resolve().then(() => task());
    
    const e: Promise<void> = p.then((result) => {
      results[index] = result;
      executing.delete(e);
    });

    executing.add(e);

    // If we hit the concurrency limit, wait for at least one to finish
    if (executing.size >= concurrency) {
      await Promise.race(executing);
    }
  }

  // Wait for all remaining active tasks to finish
  await Promise.all(executing);
  
  return results;
}

// ==========================================
// Example Usage
// ==========================================
const delay = (ms: number, id: number) => () => 
  new Promise<number>(resolve => setTimeout(() => resolve(id), ms));

// 10 tasks, each takes random time
const tasks = Array.from({ length: 10 }).map((_, i) => delay(Math.random() * 1000 + 500, i));

promisePool(tasks, 3).then(results => {
    console.log("All tasks completed!", results);
});
```
