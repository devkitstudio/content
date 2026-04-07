## Step-by-Step Memory Leak Hunting

### Step 1: Confirm the Leak
1. Open Chrome DevTools → **Memory** tab
2. Take a **Heap Snapshot** (baseline)
3. Navigate around the app (open/close pages, modals)
4. Take another **Heap Snapshot**
5. Compare: if retained size keeps growing after navigating away from pages, you have a leak

### Step 2: Use Allocation Timeline
1. Memory tab → select **Allocation instrumentation on timeline**
2. Click Start, then interact with the app
3. Look for **blue bars that never turn gray** — these are allocations that were never garbage-collected

### Step 3: Find the Retainer
1. In the Heap Snapshot comparison, sort by **Retained Size** (descending)
2. Expand the largest objects
3. Look at the **Retainers** panel → it shows the chain of references keeping the object alive
4. Common retainers:
   - `window` → global event listener
   - `setTimeout` / `setInterval` → uncleaned timer
   - Closure scope → stale closure

### Step 4: Performance Monitor
Quick visual check without deep diving:
1. DevTools → `Cmd+Shift+P` → "Show Performance Monitor"
2. Watch **JS Heap Size** and **DOM Nodes** counters
3. Navigate to a page and back — both numbers should return to roughly the same value
4. If DOM Nodes only goes up → detached DOM leak

### Quick Checklist

- Every `addEventListener` has a matching `removeEventListener` in cleanup
- Every `setInterval` / `setTimeout` is cleared in cleanup
- Every `WebSocket` / `EventSource` is closed in cleanup
- Every `fetch` uses `AbortController` with cleanup
- No module-level variables storing DOM references or large data
- State stores have eviction logic for cached data
- `useEffect` dependencies are correct (no stale closures from missing deps)
