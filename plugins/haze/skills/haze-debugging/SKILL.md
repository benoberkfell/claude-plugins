---
name: haze-debugging
description: Debug and troubleshoot Haze blur effects including platform-specific issues, performance problems, visual artifacts, and common mistakes on Android, iOS, Desktop, and Web.
---

# Debugging Haze

Troubleshooting guide for Haze blur effects across platforms.

## Common Issues

### Blur Not Appearing

**Symptoms**: No blur visible, content shows through without effect.

**Checklist**:

1. **Both modifiers present**: Need both `hazeSource` and `hazeEffect` for background blur
   ```kotlin
   // WRONG - missing hazeSource
   Box {
       LazyColumn { }
       TopAppBar(modifier = Modifier.hazeEffect(hazeState) { blurEffect { } })
   }

   // CORRECT
   Box {
       LazyColumn(modifier = Modifier.hazeSource(hazeState)) { }
       TopAppBar(modifier = Modifier.hazeEffect(hazeState) { blurEffect { } })
   }
   ```

2. **Same HazeState instance**: Both modifiers must share the same state
   ```kotlin
   // WRONG - different states
   val state1 = rememberHazeState()
   val state2 = rememberHazeState()

   // CORRECT - same state
   val hazeState = rememberHazeState()
   ```

3. **blurEffect block present**: Haze 2.0+ requires explicit effect configuration
   ```kotlin
   // WRONG in Haze 2.0+
   Modifier.hazeEffect(hazeState)

   // CORRECT
   Modifier.hazeEffect(hazeState) {
       blurEffect { style = HazeMaterials.thin() }
   }
   ```

4. **Transparent container**: The blurred element must have a transparent background
   ```kotlin
   TopAppBar(
       colors = TopAppBarDefaults.topAppBarColors(Color.Transparent),  // Required
       modifier = Modifier.hazeEffect(hazeState) { blurEffect { } }
   )
   ```

5. **Dependencies**: Ensure both `haze` and `haze-blur` modules are added

### Blur Enabled But Not Working on Android

**Android 11 and below**: Blur is disabled by default on these versions.

Enable explicitly (experimental):
```kotlin
blurEffect {
    blurEnabled = true  // Enable RenderScript-based blur
}
```

Note: RenderScript blur may appear "laggy" as it processes frames in background thread.

**Android 12-12L**: May have performance issues with progressive blur. Use mask instead:
```kotlin
progressive = HazeProgressive.verticalGradient(
    startIntensity = 1f,
    endIntensity = 0f,
    preferPerformance = true  // Falls back to mask
)
```

### Visual Artifacts

**Blur "leaking" at edges**:

Check `blurredEdgeTreatment`:
```kotlin
blurEffect {
    blurredEdgeTreatment = BlurredEdgeTreatment.Unbounded  // or .Rectangle
}
```

**Flickering or tearing**:

On Android 12 and below, the manual invalidation mechanism may cause issues:
```kotlin
blurEffect {
    // Try disabling if causing issues
    blurEnabled = false
}
```

**Noise texture issues**:

Disable noise if causing artifacts:
```kotlin
blurEffect {
    noiseFactor = 0f
}
```

## Platform-Specific Issues

### Android

| API Level | Status | Notes |
|-----------|--------|-------|
| 33+ (Android 13+) | Optimal | Full RenderEffect support |
| 31-32 (Android 12) | Good | Some progressive blur limitations |
| 30 and below | Limited | RenderScript blur (disabled by default) |

**Android 13+ Issues**:
- None expected - optimal environment

**Android 12 Issues**:
- Progressive blur limited to linear gradients only
- Other progressive types fall back to mask
- Use `preferPerformance = true` for better performance

**Android 11 and Below**:
- Blur disabled by default (scrim shown instead)
- Enable RenderScript blur with `blurEnabled = true`
- May be visually laggy due to background processing
- Only one frame processed at a time

### iOS / Desktop / Web (Skiko platforms)

These platforms use Skia-based rendering and should work optimally.

**Common issues**:
- Ensure using compatible Compose Multiplatform version
- Check that Skiko is properly bundled

### Screenshot Testing

**Robolectric testing**: Use SDK 35+ for correct edge behavior:
```kotlin
@Config(sdk = [35])
class MyScreenshotTest {
    // tests
}
```

## Performance Issues

### Measuring Performance

Use Android Macrobenchmark tests to measure Haze's impact.

**Typical overhead**:
- Simple Scaffold: ~29% increase in frame duration
- Images List: ~45% increase
- Complex draggable effects: ~98% increase

### Optimization Techniques

1. **Use Input Scale**:
   ```kotlin
   Modifier.hazeEffect(hazeState) {
       inputScale = HazeInputScale.Auto
       // or HazeInputScale.Fixed(0.5f) for 50% resolution
       blurEffect { }
   }
   ```

   Impact: 5-20% reduction in Haze cost

2. **Prefer Mask over Progressive**:
   ```kotlin
   // Faster
   blurEffect {
       mask = Brush.verticalGradient(listOf(Color.Black, Color.Transparent))
   }

   // Slower (but nicer looking)
   blurEffect {
       progressive = HazeProgressive.verticalGradient(...)
   }
   ```

3. **Disable blur when not needed**:
   ```kotlin
   blurEffect {
       blurEnabled = isVisible && !isAnimating
   }
   ```

4. **Reduce blur radius**:
   ```kotlin
   blurEffect {
       blurRadius = 12.dp  // Smaller = faster
   }
   ```

5. **Use preferPerformance on Android 12**:
   ```kotlin
   progressive = HazeProgressive.verticalGradient(
       preferPerformance = true
   )
   ```

### Frame Budget

Real-time blur is expensive. On mobile:
- Target: 16.6ms per frame (60fps)
- Haze typically adds 2-6ms
- Monitor with Android Profiler or similar tools

## Migration Issues (Haze 1.x to 2.0)

### Import Changes

```kotlin
// v1 imports - NO LONGER WORK
import dev.chrisbanes.haze.HazeStyle
import dev.chrisbanes.haze.HazeTint

// v2 imports - CORRECT
import dev.chrisbanes.haze.blur.HazeStyle
import dev.chrisbanes.haze.blur.HazeTint
import dev.chrisbanes.haze.blur.blurEffect
```

### API Changes

```kotlin
// v1 - NO LONGER WORKS
Modifier.hazeEffect(state = hazeState) {
    blurRadius = 20.dp
    tints = listOf(...)
}

// v2 - CORRECT
Modifier.hazeEffect(state = hazeState) {
    blurEffect {
        blurRadius = 20.dp
        tints = listOf(...)
    }
}
```

### Removed APIs

```kotlin
// v1 - REMOVED
val hazeState = rememberHazeState(blurEnabled = true)

// v2 - Use effect-level control
val hazeState = rememberHazeState()
blurEffect {
    blurEnabled = true
}
```

## Recursive Drawing Error

**Problem**: Using hazeSource and hazeEffect in a parent-child relationship causes recursive drawing.

```kotlin
// WRONG - will cause issues
LazyColumn(modifier = Modifier.hazeSource(hazeState)) {
    stickyHeader {
        Header(modifier = Modifier.hazeEffect(hazeState) { blurEffect { } })
    }
}

// CORRECT - use hazeSource on items instead
LazyColumn {
    stickyHeader {
        Header(modifier = Modifier.hazeEffect(hazeState) { blurEffect { } })
    }
    items(list) { item ->
        Item(modifier = Modifier.hazeSource(hazeState))
    }
}
```

## zIndex Not Working

**Problem**: Overlapping effects not drawing correct layers.

**Solution**: Ensure proper zIndex ordering:
```kotlin
Background(modifier = Modifier.hazeSource(hazeState, zIndex = 0f))
Card1(modifier = Modifier.hazeSource(hazeState, zIndex = 1f).hazeEffect(hazeState))
Card2(modifier = Modifier.hazeSource(hazeState, zIndex = 2f).hazeEffect(hazeState))
```

Effects draw all layers with zIndex less than their sibling hazeSource.

## Dialog Blur Issues

**Problem**: Tints appear as scrim over entire dialog.

**Solution**: Use translucent Surface color instead of tints:
```kotlin
Dialog(...) {
    Surface(
        color = MaterialTheme.colorScheme.surface.copy(alpha = 0.2f),  // Translucent
        modifier = Modifier.hazeEffect(hazeState) {
            blurEffect {
                style = HazeMaterials.regular()
                // Don't add extra tints for dialogs
            }
        }
    ) { }
}
```

## Memory Leaks

If you suspect memory leaks:
1. Ensure HazeState is created with `rememberHazeState()` (not `remember { HazeState() }`)
2. Check that no references to HazeState are held in long-lived objects
3. Use LeakCanary to verify

## Getting Help

- [GitHub Issues](https://github.com/chrisbanes/haze/issues)
- [GitHub Discussions](https://github.com/chrisbanes/haze/discussions)
- [Sample App](https://github.com/chrisbanes/haze/tree/main/sample) - working examples

## Diagnostic Checklist

When reporting issues, include:
1. Haze version
2. Compose version (Jetpack/Multiplatform)
3. Platform and API level (for Android)
4. Minimal reproducible code
5. Screenshots/videos if visual issue
6. Profiler traces if performance issue
