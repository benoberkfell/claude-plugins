---
name: metro-install
description: Install and configure Metro dependency injection in a Kotlin project. Use when adding Metro to a new or existing project, setting up the Gradle plugin, configuring build options, or enabling IDE support.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Metro Installation

Install and configure Metro, a compile-time dependency injection framework for Kotlin Multiplatform that combines the best of Dagger, Anvil, and kotlin-inject.

## Prerequisites

Before installing Metro, ensure the project:
- Uses Kotlin 2.2.20 or later
- Uses Gradle as the build system
- Has a compatible target: JVM, Android, JS, WASM, or Native

## Installation Steps

### Step 1: Apply the Gradle Plugin

Add the Metro Gradle plugin to the module's `build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform") // or kotlin("jvm"), kotlin("android"), etc.
    id("dev.zacsweers.metro") version "0.10.2"
}
```

The plugin automatically:
- Applies the Metro compiler plugin
- Adds the Metro runtime dependency
- Configures annotation processing

### Step 2: Configure Metro (Optional)

Configure Metro options via the `metro` extension:

```kotlin
metro {
    // Core options
    enabled = true              // Enable/disable Metro (default: true)
    debug = false               // Verbose debug logging (default: false)

    // Code generation
    generateAssistedFactories = true    // Auto-generate @AssistedFactory interfaces
    enableTopLevelFunctionInjection = true  // Enable @Inject on top-level functions

    // Interop with other DI frameworks
    interop {
        includeDagger()         // Support Dagger annotations
        includeKotlinInject()   // Support kotlin-inject annotations
        includeGuice()          // Support Guice annotations
    }

    // Build outputs
    reportsDestination = layout.buildDirectory.dir("reports/metro")
}
```

### Step 3: Create a Dependency Graph

Create your first dependency graph:

```kotlin
import dev.zacsweers.metro.DependencyGraph
import dev.zacsweers.metro.createGraph

@DependencyGraph
interface AppGraph {
    val repository: Repository
}

// Create instance
val graph = createGraph<AppGraph>()
val repo = graph.repository
```

### Step 4: Build the Project

Build to verify installation:

```bash
./gradlew :app:compileKotlin
```

## IDE Support (Recommended)

To see Metro-generated declarations in the IDE:

1. Enable K2 Mode in IntelliJ IDEA's Kotlin plugin
2. Open Registry: Help > Find Action > "Registry"
3. Set `kotlin.k2.only.bundled.compiler.plugins.enabled` to `false`
4. Restart the IDE

## Multiplatform Setup

For Kotlin Multiplatform projects:

```kotlin
plugins {
    kotlin("multiplatform")
    id("dev.zacsweers.metro") version "0.10.2"
}

kotlin {
    jvm()
    iosArm64()
    js { browser() }
    // Metro supports all Kotlin targets
}
```

## Multi-Module Setup

For multi-module projects, apply the plugin to each module that uses Metro:

```kotlin
// In each module's build.gradle.kts
plugins {
    kotlin("jvm")
    id("dev.zacsweers.metro")
}
```

Use `@ContributesTo` and `@ContributesBinding` for cross-module contributions (see metro-usage skill).

## Configuration Reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | Boolean | true | Enable/disable Metro |
| `debug` | Boolean | false | Verbose debug logging |
| `generateAssistedFactories` | Boolean | false | Auto-generate factory interfaces |
| `enableTopLevelFunctionInjection` | Boolean | false | Support top-level @Inject functions |
| `contributesAsInject` | Boolean | true | Treat @Contributes* as implicit @Inject |
| `chunkFieldInits` | Boolean | true | Split large init blocks |
| `statementsPerInitFun` | Int | 25 | Statements per init function |
| `enableGraphSharding` | Boolean | false | Distribute large graphs |
| `keysPerGraphShard` | Int | 2000 | Bindings per shard |
| `shrinkUnusedBindings` | Boolean | true | Remove unreachable bindings |

## Interop Configuration

### Migrating from Dagger

```kotlin
metro {
    interop {
        includeDagger()
        includeAnvilForDagger()  // Also support Anvil annotations
    }
}
```

### Migrating from kotlin-inject

```kotlin
metro {
    interop {
        includeKotlinInject()
    }
}
```

## Verification

After installation, verify by:
1. Building the project successfully
2. Creating a simple `@DependencyGraph` interface
3. Using `createGraph<YourGraph>()` to instantiate it

## Troubleshooting

- **Plugin not found**: Ensure the plugin is in the Gradle plugin portal or add the repository
- **Kotlin version mismatch**: Metro requires Kotlin 2.2.20+; check your Kotlin version
- **IDE not showing generated code**: Enable K2 mode and disable bundled-only plugins
- **Build errors**: Enable `debug = true` to see detailed Metro output
