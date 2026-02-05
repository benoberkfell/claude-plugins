---
name: metro-usage
description: Use Metro features for dependency injection including creating dependency graphs, adding bindings, scopes, providers, multibindings, assisted injection, and multi-module aggregation.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Metro Usage

Use Metro's dependency injection features to wire up your application. This covers dependency graphs, bindings, scopes, providers, and multi-module aggregation.

## Core Concepts

### Dependency Graphs

The `@DependencyGraph` annotation marks the entry point for injection:

```kotlin
import dev.zacsweers.metro.DependencyGraph
import dev.zacsweers.metro.createGraph

@DependencyGraph
interface AppGraph {
    // Accessors expose bindings
    val repository: Repository
    val apiClient: ApiClient
}

// Create and use
val graph = createGraph<AppGraph>()
val repo = graph.repository
```

### Constructor Injection

Mark classes with `@Inject` to enable constructor injection:

```kotlin
import dev.zacsweers.metro.Inject

@Inject
class Repository(
    private val database: Database,
    private val apiClient: ApiClient
)

@Inject
class Database(private val config: Config)

@Inject
class ApiClient(private val httpClient: HttpClient)
```

### Provider Functions

Use `@Provides` to create bindings for types you don't control:

```kotlin
@DependencyGraph
interface AppGraph {
    val httpClient: HttpClient

    @Provides
    fun provideHttpClient(): HttpClient = HttpClient()

    @Provides
    fun provideConfig(): Config = Config(baseUrl = "https://api.example.com")
}
```

Providers can also be in supertypes:

```kotlin
interface NetworkProviders {
    @Provides fun provideHttpClient(): HttpClient = HttpClient()
}

@DependencyGraph
interface AppGraph : NetworkProviders {
    val httpClient: HttpClient
}
```

### Binds (Interface Binding)

Use `@Binds` to bind an implementation to its interface:

```kotlin
@DependencyGraph
interface AppGraph {
    val repository: Repository

    @Binds val RepositoryImpl.bind: Repository

    @Provides fun provideImpl(): RepositoryImpl = RepositoryImpl()
}
```

## Scopes

### Scoping Bindings

Use `@SingleIn` to ensure only one instance exists per graph:

```kotlin
import dev.zacsweers.metro.SingleIn

// Define a scope marker
abstract class AppScope private constructor()

@SingleIn(AppScope::class)
@Inject
class Database(private val config: Config)

// Graph must declare the scope
@DependencyGraph(AppScope::class)
interface AppGraph {
    val database: Database  // Same instance on every access
}
```

### Scope Rules

- Scoped bindings can only be used in graphs that declare their scope
- An unscoped graph cannot access scoped bindings (compile error)
- Multiple scopes can be combined with `additionalScopes`

```kotlin
@DependencyGraph(
    scope = AppScope::class,
    additionalScopes = [FeatureScope::class]
)
interface AppGraph
```

## Qualifiers

Use qualifiers to distinguish bindings of the same type:

```kotlin
import dev.zacsweers.metro.Named

@DependencyGraph
interface AppGraph {
    @Named("api") val apiUrl: String
    @Named("db") val dbUrl: String

    @Provides @Named("api")
    fun provideApiUrl(): String = "https://api.example.com"

    @Provides @Named("db")
    fun provideDbUrl(): String = "jdbc:mysql://localhost/db"
}
```

Create custom qualifiers:

```kotlin
import dev.zacsweers.metro.Qualifier

@Qualifier
annotation class ApiBaseUrl

@DependencyGraph
interface AppGraph {
    @ApiBaseUrl val baseUrl: String

    @Provides @ApiBaseUrl
    fun provideBaseUrl(): String = "https://api.example.com"
}
```

## Multibindings

### Set Multibindings

Collect multiple bindings into a Set:

```kotlin
import dev.zacsweers.metro.IntoSet

@DependencyGraph
interface AppGraph {
    val handlers: Set<Handler>

    @Provides @IntoSet
    fun provideHandler1(): Handler = LoggingHandler()

    @Provides @IntoSet
    fun provideHandler2(): Handler = MetricsHandler()
}
```

### Map Multibindings

Collect bindings into a Map:

```kotlin
import dev.zacsweers.metro.IntoMap
import dev.zacsweers.metro.StringKey

@DependencyGraph
interface AppGraph {
    val handlers: Map<String, Handler>

    @Provides @IntoMap @StringKey("logging")
    fun provideLoggingHandler(): Handler = LoggingHandler()

    @Provides @IntoMap @StringKey("metrics")
    fun provideMetricsHandler(): Handler = MetricsHandler()
}
```

### Empty Multibindings

Allow empty collections with `@Multibinds`:

```kotlin
@DependencyGraph
interface AppGraph {
    @Multibinds(allowEmpty = true)
    val plugins: Set<Plugin>
}
```

## Graph Factories

### Runtime Parameters

Use factories for runtime inputs:

```kotlin
@DependencyGraph
interface UserGraph {
    val userRepository: UserRepository

    @DependencyGraph.Factory
    fun interface Factory {
        fun create(@Provides userId: String): UserGraph
    }
}

// Usage
val factory = createGraphFactory<UserGraph.Factory>()
val userGraph = factory.create("user-123")
```

### Including Other Graphs

Compose graphs with `@Includes`:

```kotlin
@DependencyGraph
interface AppGraph {
    val config: Config

    @Provides fun provideConfig(): Config = Config()
}

@DependencyGraph
interface FeatureGraph {
    val feature: Feature

    @DependencyGraph.Factory
    fun interface Factory {
        fun create(@Includes appGraph: AppGraph): FeatureGraph
    }
}
```

## Assisted Injection

For classes needing both injected and runtime parameters:

```kotlin
import dev.zacsweers.metro.AssistedInject
import dev.zacsweers.metro.Assisted
import dev.zacsweers.metro.AssistedFactory

@AssistedInject
class HttpClient(
    @Assisted val timeout: Duration,  // Runtime parameter
    val cache: Cache                   // Injected
) {
    @AssistedFactory
    fun interface Factory {
        fun create(timeout: Duration): HttpClient
    }
}

// Usage
@Inject
class ApiClient(private val httpClientFactory: HttpClient.Factory) {
    private val client = httpClientFactory.create(30.seconds)
}
```

Enable auto-generation of factories:

```kotlin
metro {
    generateAssistedFactories = true
}
```

## Multi-Module Aggregation

### Contributing to Scopes

Use `@ContributesTo` to contribute provider interfaces across modules:

```kotlin
// In :network module
@ContributesTo(AppScope::class)
interface NetworkProviders {
    @Provides fun provideHttpClient(): HttpClient = HttpClient()
}

// In :app module - automatically includes NetworkProviders
@DependencyGraph(AppScope::class)
interface AppGraph {
    val httpClient: HttpClient
}
```

### Contributing Bindings

Use `@ContributesBinding` to bind implementations across modules:

```kotlin
// In :impl module
@ContributesBinding(AppScope::class)
@Inject
class RepositoryImpl(private val api: Api) : Repository

// In :app module - Repository automatically resolves to RepositoryImpl
@DependencyGraph(AppScope::class)
interface AppGraph {
    val repository: Repository
}
```

### Contributing to Multibindings

```kotlin
@ContributesIntoSet(AppScope::class)
@Inject
class AnalyticsHandler : Handler

@ContributesIntoMap(AppScope::class)
@StringKey("feature")
@Inject
class FeatureHandler : Handler
```

### Replacements (Testing)

Replace contributions in tests:

```kotlin
// Replace in test graph
@ContributesTo(AppScope::class, replaces = [RealNetworkProviders::class])
interface FakeNetworkProviders {
    @Provides fun provideFakeClient(): HttpClient = FakeHttpClient()
}

// Or exclude in graph
@DependencyGraph(AppScope::class, excludes = [RealNetworkProviders::class])
interface TestAppGraph
```

## Provider and Lazy Types

Defer creation or break dependency cycles:

```kotlin
import dev.zacsweers.metro.Provider
import kotlin.Lazy

@Inject
class ApiClient(
    private val cache: Provider<Cache>,  // Created on each call
    private val db: Lazy<Database>       // Created on first access
) {
    fun fetch() {
        val c = cache()        // New instance (or same if scoped)
        val d = db.value       // Lazy initialization
    }
}
```

## Graph Extensions

Create child graphs that extend parent graphs:

```kotlin
@GraphExtension(LoggedInScope::class)
interface LoggedInGraph {
    val user: User
    val userRepository: UserRepository

    @ContributesTo(AppScope::class)
    @GraphExtension.Factory
    interface Factory {
        fun create(@Provides userId: String): LoggedInGraph
    }
}

// AppGraph automatically implements the factory
@DependencyGraph(AppScope::class)
interface AppGraph

// Usage
val appGraph = createGraph<AppGraph>()
val loggedInGraph = appGraph.asContribution<LoggedInGraph.Factory>()
    .create("user-123")
```

## Member Injection

Inject into existing instances (e.g., Android Activities):

```kotlin
class MainActivity : Activity() {
    @Inject lateinit var viewModel: MainViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        appGraph.inject(this)
    }
}

@DependencyGraph(AppScope::class)
interface AppGraph {
    fun inject(activity: MainActivity)
}
```

## Binding Containers

Reusable modules with configurable bindings:

```kotlin
@BindingContainer
class NetworkBindings(private val baseUrl: String) {
    @Provides fun provideHttpClient(): HttpClient = HttpClient(baseUrl)
}

@DependencyGraph
interface AppGraph {
    val httpClient: HttpClient

    @DependencyGraph.Factory
    interface Factory {
        fun create(@Includes networkBindings: NetworkBindings): AppGraph
    }
}

// Usage
val graph = createGraphFactory<AppGraph.Factory>()
    .create(NetworkBindings("https://api.example.com"))
```

## Optional Bindings

Handle missing dependencies with defaults:

```kotlin
@DependencyGraph
interface AppGraph {
    // Parameter with default value - uses default if binding missing
    @Provides
    fun provideMessage(count: Int = -1): String = "Count: $count"

    // Optional accessor
    @OptionalBinding
    val debugFlag: Boolean? get() = null
}
```
