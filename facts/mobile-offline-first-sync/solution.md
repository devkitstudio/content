## Offline-First Architecture

Three main conflict resolution strategies:

### 1. Last-Write-Wins (LWW) - Simplest

```typescript
interface SyncRecord {
  id: string;
  data: any;
  timestamp: number; // milliseconds
  clientId: string;
}

// Conflict resolution: keep latest write
function resolveConflict(local: SyncRecord, remote: SyncRecord): SyncRecord {
  return local.timestamp > remote.timestamp ? local : remote;
}

// Usage
const localEdit = { id: '123', data: { name: 'Alice' }, timestamp: 1000 };
const remoteEdit = { id: '123', data: { name: 'Bob' }, timestamp: 2000 };
const resolved = resolveConflict(localEdit, remoteEdit);
// Result: remoteEdit (later timestamp wins)
```

**Pros:** Simple, low overhead
**Cons:** Data loss (earlier edits discarded)

### 2. CRDT (Conflict-free Replicated Data Type) - Production-Grade

```typescript
// Yjs CRDT implementation
import * as Y from 'yjs';
import { IndexeddbPersistence } from 'y-indexeddb';

class OfflineSyncManager {
  ydoc = new Y.Doc();
  persistence = new IndexeddbPersistence('app-db', this.ydoc);
  ymap = this.ydoc.getMap('shared-state');

  // Any user can write; CRDT automatically resolves conflicts
  updateRecord(key: string, value: any) {
    this.ymap.set(key, value);
  }

  // Get current merged state
  getRecord(key: string) {
    return this.ymap.get(key);
  }

  // Sync with server
  async syncToServer() {
    const update = Y.encodeStateAsUpdate(this.ydoc);
    await fetch('/api/sync', {
      method: 'POST',
      body: update,
    });
  }

  // Receive updates from server
  mergeServerUpdates(update: Uint8Array) {
    Y.applyUpdate(this.ydoc, update);
  }
}

// Usage
const manager = new OfflineSyncManager();
manager.updateRecord('user-profile', { name: 'Alice', age: 30 });

// Even if both clients edit offline:
// Client A: { name: 'Alice', age: 31 }
// Client B: { name: 'Alice', age: 32 }
// After sync: CRDT merges them intelligently
```

**Pros:** No data loss, automatic merging, works offline
**Cons:** Requires specialized data structures

### 3. Three-Way Merge (Git-Style)

```typescript
interface MergeSnapshot {
  base: Record<string, any>;    // Last known common state
  local: Record<string, any>;   // Our changes
  remote: Record<string, any>;  // Server changes
}

function threeWayMerge(snapshot: MergeSnapshot): {
  merged: Record<string, any>;
  conflicts: string[];
} {
  const conflicts: string[] = [];
  const merged: Record<string, any> = { ...snapshot.base };

  // For each field, resolve independently
  const allKeys = new Set([
    ...Object.keys(snapshot.local),
    ...Object.keys(snapshot.remote),
  ]);

  for (const key of allKeys) {
    const baseVal = snapshot.base[key];
    const localVal = snapshot.local[key];
    const remoteVal = snapshot.remote[key];

    // No conflict: one side changed
    if (localVal === remoteVal) {
      merged[key] = localVal ?? remoteVal;
    }
    // Both sides changed differently
    else if (localVal !== baseVal && remoteVal !== baseVal) {
      conflicts.push(key);
      merged[key] = remoteVal; // Default to server, user can retry
    }
    // Only local changed
    else if (localVal !== baseVal) {
      merged[key] = localVal;
    }
    // Only remote changed
    else {
      merged[key] = remoteVal;
    }
  }

  return { merged, conflicts };
}

// Usage
const result = threeWayMerge({
  base: { title: 'Meeting', time: '9am' },
  local: { title: 'Team Sync', time: '9am' }, // User changed title
  remote: { title: 'Meeting', time: '10am' }, // Server changed time
});
// Result: { merged: { title: 'Team Sync', time: '10am' }, conflicts: [] }
```

**Pros:** Fine-grained conflict detection, no automatic overwrites
**Cons:** More complex, needs base snapshot

## Operational Implementation

```typescript
class SyncEngine {
  db: LocalDatabase;
  syncQueue: Operation[] = [];

  async queueOperation(op: Operation) {
    // Store locally first
    this.syncQueue.push(op);
    this.db.execute(op);

    // Try to sync if online
    if (navigator.onLine) {
      await this.syncToServer();
    }
  }

  async syncToServer() {
    while (this.syncQueue.length > 0) {
      const op = this.syncQueue[0];
      try {
        const response = await fetch('/api/sync', {
          method: 'POST',
          body: JSON.stringify(op),
        });

        if (response.ok) {
          this.syncQueue.shift();
        } else if (response.status === 409) {
          // Conflict: merge with server state
          const serverState = await response.json();
          const resolved = this.resolveConflict(op, serverState);
          this.db.execute(resolved);
          this.syncQueue[0] = resolved;
        }
      } catch (e) {
        // Network error: retry later
        break;
      }
    }
  }

  resolveConflict(local: Operation, remote: Operation): Operation {
    // Use CRDT or LWW
    return remote.timestamp > local.timestamp ? remote : local;
  }
}

// Usage
const engine = new SyncEngine();

// Works offline
await engine.queueOperation({ action: 'UPDATE_PROFILE', data: {...} });

// Automatically syncs when online
window.addEventListener('online', () => engine.syncToServer());
```
