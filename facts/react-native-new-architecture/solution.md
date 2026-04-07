## New Architecture Overview

The old bridge was synchronous and serialized all calls. New Architecture uses three key improvements:

### 1. Fabric (UI Layer)
- **Direct thread access** to native UI without serialization delays
- **Synchronous layout measurement** (critical for complex animations)
- Eliminates the bridge for every UI update

```kotlin
// Fabric Native Module (Kotlin)
object FabricModule : ReactContextBaseJavaModule {
  override fun getName() = "FabricModule"

  @ReactMethod
  fun measureText(text: String, promise: Promise) {
    // Direct access without bridge serialization
    val paint = TextPaint()
    val width = paint.measureText(text)
    promise.resolve(width)
  }
}
```

### 2. TurboModules
- **Eager module initialization** with typed interfaces
- **Native method lookup at startup** (no runtime reflection)
- **Direct C++ binding** instead of JSON serialization

```typescript
// TurboModule definition (TypeScript)
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  fetchUserData(userId: string): Promise<User>;
  getCameraFrame(): number; // Returns frame pointer
}

export default TurboModuleRegistry.getEnforcing<Spec>(
  'NativeUserModule'
);
```

```swift
// Native implementation (Swift)
@objc(RCT_EXTERN_MODULE(NativeUserModule, NSObject))
class NativeUserModule: NSObject {
  @objc func fetchUserData(_ userId: String) -> Promise {
    return Promise { resolve, reject in
      API.fetchUser(userId) { user, error in
        if let error = error {
          reject("FETCH_ERROR", error.localizedDescription, error)
        } else {
          resolve(user?.toDictionary())
        }
      }
    }
  }

  @objc func getCameraFrame() -> NSNumber {
    let pointer = getCameraFrame()
    return NSNumber(value: pointer)
  }

  @objc static func requiresMainQueueSetup() -> Bool {
    return true
  }
}
```

### 3. Eliminating Bridge Bottleneck

**Old Bridge (React Native 0.62-):**
- JS → JSON serialization → Bridge → Native
- JS → Bridge → JSON → Native (round-trip for each call)
- Synchronous waits block JS thread

**New Architecture:**
- JSI (JavaScript Interface) for direct C++ calls
- No serialization overhead
- Asynchronous by default, synchronous when needed

```javascript
// Using JSI for direct access
import { NativeModules } from 'react-native';

// Old way (JSON serialization)
NativeModules.Camera.captureFrame(frameData); // Serializes entire frame

// New way (JSI - direct memory access)
const nativeCamera = global.nativeCamera; // Direct C++ pointer
nativeCamera.captureFrameSync(framePointer); // No serialization
```

## Performance Gains

| Metric | Old Bridge | New Architecture |
|--------|-----------|------------------|
| Module init | ~100ms | ~10ms |
| Native call | 5-10ms | 0.5-2ms |
| Large data transfer | ~500ms | ~50ms |
| Render pass | 16.67ms (60fps bottleneck) | Passes consistently |

## Migration Path

1. Enable Fabric in `android/gradle.properties`
2. Rewrite custom modules as TurboModules
3. Use JSI for performance-critical paths
