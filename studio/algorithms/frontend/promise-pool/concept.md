

# Promise Concurrency Limiter (Promise Pool)

**Promise Concurrency Limiter** is an essential pattern in Frontend development and NodeJS used to execute a large backlog of asynchronous tasks (like fetching APIs, reading files, or downloading images) without overloading the target server or running out of memory. 

Instead of firing 1000 requests using `Promise.all` which might cause a 429 Too Many Requests error or crash the browser network queue, a concurrency limiter strictly enforces that only a maximum of `N` tasks run continuously at any given time.

## How it Works
1. **The Queue**: You maintain a queue of tasks waiting to go.
2. **The Pool**: An array (or Set) of currently executing promises.
3. **Execution**:
   - Loop through the tasks.
   - For each task, add it to the Pool.
   - If the Pool size reaches the concurrency limit `N`, wait for the fastest task to resolve using `Promise.race(pool)`.
   - Once a task finishes, remove it from the pool, freeing up a slot for the next task in the queue.
4. **Completion**: Finally, wait for the remaining tasks in the pool to resolve using `Promise.all`.

## Why is this important?
In Senior Frontend Interviews, writing a reliable Promise Pool from scratch is one of the most common questions. It tests a developer's understanding of the JavaScript event loop, microtask queues, closure retention, and the deep mechanics of Promises beyond simple `async/await`.
