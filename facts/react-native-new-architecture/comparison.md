## Architecture Comparison

### Old Architecture (Pre-0.68)

```
JS Thread          Bridge              Native Thread
┌─────────────┐   ┌──────────────┐   ┌──────────────┐
│ JS Code     │→→→│ JSON Serial  │→→→│ Native Code  │
│ (runs)      │   │ (expensive)  │   │ (executes)   │
│ Waits here  │←←←│ JSON Deserial│←←←│              │
└─────────────┘   └──────────────┘   └──────────────┘
     ↓
  BLOCKED on
  native call
```

**Problems:**
- Synchronous waits block UI thread
- JSON serialization overhead (5-10ms per call)
- Reflection for method lookup at runtime
- Module initialization happens lazily (first use)

### New Architecture (0.68+)

```
JS Thread (JSI)         Native Threads
┌────────────────────────────────────┐
│ Direct C++ Function Pointers        │
│ No serialization, no wait           │
│ JSI provides typed interface        │
└────────────────────────────────────┘
         ↓
  Fabric (UI)      TurboModules (Logic)
  ┌─────────────┐  ┌──────────────────┐
  │ Synchronous │  │ Async by default  │
  │ layout      │  │ with sync option  │
  │ measurement │  │ Direct C++ calls  │
  └─────────────┘  └──────────────────┘
```

**Benefits:**
- JSI eliminates serialization
- TurboModules use eager initialization (all loaded at startup)
- Fabric enables synchronous layout ops
- Method lookup happens once at init

## Call Timing Comparison

**Old Bridge - Heavy Serialization:**
```javascript
const data = { userId: 123, items: [...Array(100)] };
NativeModules.API.sendData(data);
// Timeline:
// - JS → JSON.stringify: 2ms
// - Bridge marshalling: 3ms
// - Native parsing: 2ms
// Total: 7ms overhead per call
```

**New Architecture - Direct Call:**
```typescript
const api = TurboModuleRegistry.getEnforcing<APISpec>('API');
await api.sendData(data);
// Timeline:
// - Direct C++ call: 0.2ms
// - No serialization
// - Total overhead: 0.2ms
```

## Module Initialization

**Old:** Modules loaded on-demand (first call may have 50-100ms delay)

**New:** All TurboModules loaded at app startup (deterministic ~300ms total)

## When to Use Which

| Scenario | Old | New |
|----------|-----|-----|
| Simple string/number calls | Good | Better |
| Large data transfer | Slow | Fast |
| Frequent polling (100+ calls/sec) | Bottleneck | Efficient |
| Synchronous layout ops | Impossible | Native |
| Custom native modules | Moderate friction | Lower friction |
