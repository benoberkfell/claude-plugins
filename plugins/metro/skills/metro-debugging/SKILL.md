---
name: metro-debugging
description: Debug and troubleshoot Metro dependency injection issues including compile errors, missing bindings, dependency cycles, scope mismatches, and runtime problems. Use for graph visualization and analysis.
allowed-tools: Read, Glob, Grep, Edit, Write, Bash
---

# Metro Debugging

Debug and troubleshoot Metro dependency injection issues. This covers compile-time errors, graph visualization, performance analysis, and common problems.

## Debug Logging

Enable verbose debug output to see generated code structure:

```kotlin
metro {
    debug = true
}
```

This prints pseudocode for generated IR classes during compilation.

## Graph Reports and Visualization

### Generate Reports

Configure report output:

```kotlin
metro {
    reportsDestination = layout.buildDirectory.dir("reports/metro")
}
```

### Generate and View

```bash
# Generate graph metadata
./gradlew :app:generateMetroGraphMetadata

# Analyze and produce metrics
./gradlew :app:analyzeMetroGraph

# Generate interactive HTML visualization
./gradlew :app:generateMetroGraphHtml
# Opens automatically in browser
```

### Understanding the Visualization

**Node Colors (binding types):**
- **Blue**: Constructor-injected (`@Inject`)
- **Green**: Provided (`@Provides`)
- **Gray**: Alias (`@Binds`)
- **Light Blue**: Bound instance
- **Pink**: Multibinding
- **Purple**: Graph extension
- **Peach**: Assisted injection

**Node Shapes:**
- **Diamond**: Main `@DependencyGraph`
- **Rounded rectangle**: `@GraphExtension`
- **Magenta border circle**: Scoped binding (`@SingleIn`)
- **Regular circle**: Normal binding

**Edge Styles:**
- **Gray solid**: Normal dependency
- **Light blue thick**: Accessor (entry point)
- **Magenta dashed**: Inherited binding
- **Cyan dashed**: Deferrable (`Provider`/`Lazy`)
- **Orange thick**: Assisted injection
- **Purple**: Multibinding contribution
- **Gray dotted**: Alias

**Glow Effects (metrics):**
- **Red glow**: High betweenness centrality (critical path)
- **Yellow glow**: Moderate centrality
- **Blue glow**: High fan-in (many dependents)

## Common Compile-Time Errors

### Missing Binding

**Error:**
```
[Metro/MissingBinding] Cannot find @Inject constructor or @Provides for: kotlin.Int

    kotlin.Int is requested at
        [test.ExampleGraph] test.ExampleGraph.int

Similar bindings:
  - @Named("qualified") Int (Different qualifier)
  - Number (Supertype)
```

**Solutions:**
1. Add `@Inject` to the class constructor:
   ```kotlin
   @Inject
   class MyClass(...)
   ```

2. Add a `@Provides` function:
   ```kotlin
   @Provides fun provideInt(): Int = 42
   ```

3. Check qualifier mismatch - the error shows similar bindings with different qualifiers

4. For multi-module: use `@ContributesBinding` in the implementation module

### Dependency Cycle

**Error:**
```
[Metro/DependencyCycle] Found a dependency cycle:
  kotlin.Int → kotlin.String → kotlin.Double → kotlin.Int
```

**Solutions:**
1. Use `Provider<T>` to defer one dependency:
   ```kotlin
   @Inject
   class A(private val bProvider: Provider<B>) {
       fun useB() = bProvider()
   }
   ```

2. Use `Lazy<T>` for lazy initialization:
   ```kotlin
   @Inject
   class A(private val b: Lazy<B>) {
       fun useB() = b.value
   }
   ```

3. Restructure dependencies to break the cycle

### Scope Mismatch

**Error:**
```
[Metro/ScopeMismatch] Scoped binding 'Database' with scope 'AppScope'
cannot be used in unscoped graph 'FeatureGraph'
```

**Solutions:**
1. Add the scope to the graph:
   ```kotlin
   @DependencyGraph(AppScope::class)
   interface FeatureGraph
   ```

2. Remove the scope from the binding if it shouldn't be scoped

3. Include the scoped graph as a dependency:
   ```kotlin
   @DependencyGraph.Factory
   interface Factory {
       fun create(@Includes appGraph: AppGraph): FeatureGraph
   }
   ```

### Duplicate Binding

**Error:**
```
[Metro/DuplicateBinding] Multiple bindings found for: Repository
  - RepositoryImpl via @ContributesBinding
  - AnotherRepositoryImpl via @ContributesBinding
```

**Solutions:**
1. Use `replaces` to explicitly choose one:
   ```kotlin
   @ContributesBinding(AppScope::class, replaces = [AnotherRepositoryImpl::class])
   class RepositoryImpl : Repository
   ```

2. Use qualifiers to distinguish them:
   ```kotlin
   @Named("primary") @ContributesBinding(AppScope::class)
   class RepositoryImpl : Repository
   ```

3. Use `excludes` in the graph:
   ```kotlin
   @DependencyGraph(AppScope::class, excludes = [AnotherRepositoryImpl::class])
   interface AppGraph
   ```

### Unmatched Exclusions/Replacements

When `reportsDestination` is set, check these files:
- `merging-unmatched-exclusions-*.txt`
- `merging-unmatched-replacements-*.txt`

These indicate `excludes` or `replaces` targeting non-existent contributions.

## Runtime Debugging

### Dynamic Graphs for Testing

Replace bindings at runtime:

```kotlin
@DependencyGraph
interface AppGraph {
    val repository: Repository
    @Provides fun provideRepository(): Repository = RealRepository()
}

@BindingContainer
object FakeBindings {
    @Provides fun provideRepository(): Repository = FakeRepository()
}

class AppTest {
    @Test
    fun testWithFake() {
        val graph = createDynamicGraph<AppGraph>(FakeBindings)
        // graph.repository now returns FakeRepository
        assertIs<FakeRepository>(graph.repository)
    }
}
```

### Inspecting Generated Code

For JVM targets, decompile generated code:

1. Navigate to build output: `build/classes/kotlin/main/`
2. Find the generated graph implementation (e.g., `Metro_AppGraph.class`)
3. Use IntelliJ's "Decompile to Java" action

For Android: check `build/tmp/kotlin-classes/debug/`

## Performance Debugging

### Compilation Tracing

Generate Perfetto traces:

```kotlin
metro {
    traceDestination = layout.buildDirectory.dir("metro/trace")
}
```

Build with cache bypass:
```bash
./gradlew :app:compileKotlin --rerun
```

Load the trace in [Perfetto UI](https://ui.perfetto.dev).

### Build Performance Issues

**Slow incremental builds:**
- Enable graph sharding for large graphs:
  ```kotlin
  metro {
      enableGraphSharding = true
      keysPerGraphShard = 2000
  }
  ```

**Large method errors (JVM):**
- Enable chunking (default):
  ```kotlin
  metro {
      chunkFieldInits = true
      statementsPerInitFun = 25
  }
  ```

**Slow initialization:**
- Enable switching providers for deferred class loading:
  ```kotlin
  metro {
      enableSwitchingProviders = true
  }
  ```

## Validation Commands

### Validate Graph Structure

```bash
# Full build with validation
./gradlew :app:compileKotlin

# Check for unused bindings (enabled by default)
# Warnings appear if shrinkUnusedBindings detects unreachable bindings
```

### Check Contributions

Verify that contributions are being aggregated:

1. Enable debug mode temporarily
2. Check that `@ContributesTo` interfaces appear in generated graph
3. Verify scope markers match between contributions and graph

## Troubleshooting Checklist

### Binding Not Found
- [ ] Class has `@Inject` on constructor?
- [ ] `@Provides` function exists?
- [ ] Qualifiers match exactly?
- [ ] Scope is accessible from the graph?
- [ ] Multi-module: `@ContributesBinding` has correct scope?

### Graph Not Generating
- [ ] Metro plugin applied in build.gradle.kts?
- [ ] Kotlin version is 2.2.20+?
- [ ] `metro.enabled` is not set to `false`?
- [ ] Interface/class has `@DependencyGraph` annotation?

### Contributions Not Merging
- [ ] Scope markers match?
- [ ] Plugin applied to contributing module?
- [ ] No typos in scope class references?
- [ ] Contributing module is a dependency of the graph module?

### IDE Not Showing Generated Code
- [ ] K2 mode enabled?
- [ ] `kotlin.k2.only.bundled.compiler.plugins.enabled` set to `false`?
- [ ] Project rebuilt after changes?

## Diagnostic Configuration

Full diagnostic setup:

```kotlin
metro {
    debug = true
    reportsDestination = layout.buildDirectory.dir("reports/metro")
    traceDestination = layout.buildDirectory.dir("metro/trace")
}
```

Then inspect:
- Build output for debug logs
- `build/reports/metro/` for graph reports
- `build/metro/trace/` for performance traces

## Getting Help

If issues persist:

1. Check the [Metro documentation](https://zacsweers.github.io/metro/)
2. Search [GitHub issues](https://github.com/ZacSweers/metro/issues)
3. Create a minimal reproduction case
4. Enable `debug = true` and include relevant output
