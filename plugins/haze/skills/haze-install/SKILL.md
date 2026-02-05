---
name: haze-install
description: Install and configure Haze blur effects library in a Compose Multiplatform project. Use when adding Haze to a new or existing project, setting up dependencies, or configuring for different platforms.
---

# Installing Haze

Haze is a library providing visual effects (blur, glassmorphism) for Compose Multiplatform.

## Platform Support

| Platform      | Supported |
|---------------|-----------|
| Android       | Yes       |
| Desktop (JVM) | Yes       |
| iOS           | Yes       |
| Wasm          | Yes       |
| JS/Canvas     | Yes       |

## Dependencies

Add dependencies in your `build.gradle.kts`:

```kotlin
repositories {
    mavenCentral()
}

dependencies {
    // Core library infrastructure (required)
    implementation("dev.chrisbanes.haze:haze:1.7.1")

    // Blur effect module (required for blur effects)
    implementation("dev.chrisbanes.haze:haze-blur:1.7.1")

    // Optional: Pre-built blur styles (Material, Cupertino, Fluent)
    implementation("dev.chrisbanes.haze:haze-materials:1.7.1")
}
```

### Module Architecture (Haze 2.0+)

Haze uses a modular architecture:

- **haze** - Core infrastructure (`HazeState`, modifiers, `VisualEffect` interface)
- **haze-blur** - Blur effect implementation
- **haze-materials** - Pre-built blur styles (HazeMaterials, CupertinoMaterials, FluentMaterials)

The core `haze` module alone only provides infrastructure. To use blur effects, you need `haze-blur`.

## Kotlin Multiplatform Setup

For a multiplatform project, add dependencies in `commonMain`:

```kotlin
// build.gradle.kts
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("dev.chrisbanes.haze:haze:1.7.1")
            implementation("dev.chrisbanes.haze:haze-blur:1.7.1")
            implementation("dev.chrisbanes.haze:haze-materials:1.7.1")
        }
    }
}
```

## Version Compatibility

Haze is built and tested against the latest stable Compose Multiplatform version.

### Key Dependencies for 1.7.x
- Kotlin 2.2.20
- Compose Multiplatform 1.9.3
- Jetpack Compose 1.9.4

## Android-Specific Configuration

### Minimum SDK Requirements

- Minimum SDK: API 21
- Optimal experience: API 33+ (Android 13+)
- Full blur support on API 12+ (with some limitations on older versions)

### Compile SDK

The library targets the latest compile SDK. If you see warnings:

```kotlin
// gradle.properties
android.suppressUnsupportedCompileSdk=34
```

## Using Snapshot Versions

For latest development features:

```kotlin
repositories {
    maven("https://oss.sonatype.org/content/repositories/snapshots/")
}

dependencies {
    implementation("dev.chrisbanes.haze:haze:2.0.0-SNAPSHOT")
    implementation("dev.chrisbanes.haze:haze-blur:2.0.0-SNAPSHOT")
}
```

## Imports

After adding dependencies, import the required classes:

```kotlin
// Core
import dev.chrisbanes.haze.HazeState
import dev.chrisbanes.haze.rememberHazeState
import dev.chrisbanes.haze.hazeSource
import dev.chrisbanes.haze.hazeEffect

// Blur effect (Haze 2.0+)
import dev.chrisbanes.haze.blur.blurEffect
import dev.chrisbanes.haze.blur.HazeStyle
import dev.chrisbanes.haze.blur.HazeTint
import dev.chrisbanes.haze.blur.HazeProgressive

// Materials (optional)
import dev.chrisbanes.haze.blur.materials.HazeMaterials
import dev.chrisbanes.haze.blur.materials.CupertinoMaterials
import dev.chrisbanes.haze.blur.materials.FluentMaterials
```

## Verification

Test your setup with a minimal example:

```kotlin
@Composable
fun HazeTest() {
    val hazeState = rememberHazeState()

    Box {
        Box(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.Blue)
                .hazeSource(state = hazeState)
        )

        Box(
            modifier = Modifier
                .align(Alignment.Center)
                .size(200.dp)
                .hazeEffect(state = hazeState) {
                    blurEffect {
                        blurRadius = 20.dp
                    }
                }
        ) {
            Text("Blurred!", modifier = Modifier.align(Alignment.Center))
        }
    }
}
```

## Next Steps

- See the [haze-usage](/haze-usage) skill for detailed usage patterns
- See the [haze-debugging](/haze-debugging) skill for troubleshooting
