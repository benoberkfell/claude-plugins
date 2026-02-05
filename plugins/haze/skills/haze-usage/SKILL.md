---
name: haze-usage
description: Use Haze features for blur effects in Compose Multiplatform. Covers hazeSource, hazeEffect, blurEffect, styling, progressive blurs, masking, overlapping effects, and common UI patterns.
---

# Using Haze

Haze provides blur (glassmorphism) effects for Compose Multiplatform using two main modifiers: `hazeSource` and `hazeEffect`.

## Core Concepts

### HazeState

A state holder that manages blur rendering. Share between source and effect:

```kotlin
val hazeState = rememberHazeState()
```

### hazeSource vs hazeEffect

- **hazeSource**: Marks content that can be blurred by effects elsewhere
- **hazeEffect**: Applies the blur effect, drawing blurred content from marked sources

## Basic Background Blur

The most common pattern - blur content behind a UI element:

```kotlin
val hazeState = rememberHazeState()

Box {
    // Content to blur (e.g., scrolling list)
    LazyColumn(
        modifier = Modifier
            .fillMaxSize()
            .hazeSource(state = hazeState)
    ) {
        items(100) { Text("Item $it") }
    }

    // Blurred app bar
    TopAppBar(
        colors = TopAppBarDefaults.topAppBarColors(Color.Transparent),
        modifier = Modifier
            .hazeEffect(state = hazeState) {
                blurEffect {
                    style = HazeMaterials.thin()
                }
            }
    )
}
```

## Foreground Blur

Blur content within a composable itself (no HazeState needed):

```kotlin
Box(
    modifier = Modifier
        .hazeEffect {
            blurEffect {
                blurRadius = 20.dp
            }
        }
) {
    // This content will be blurred
    Image(painter = painterResource(id = R.drawable.photo), ...)
}
```

## Blur Styling

### Using Pre-built Materials

```kotlin
// Material Design styles
blurEffect { style = HazeMaterials.thin() }
blurEffect { style = HazeMaterials.regular() }
blurEffect { style = HazeMaterials.thick() }

// iOS/Apple styles
blurEffect { style = CupertinoMaterials.thin() }
blurEffect { style = CupertinoMaterials.regular() }

// Windows/Fluent styles
blurEffect { style = FluentMaterials.thin() }
```

### Custom Styling

Set properties directly in `blurEffect {}`:

```kotlin
blurEffect {
    blurRadius = 25.dp           // Blur strength
    tints = listOf(              // Color overlays
        HazeTint(Color.Black.copy(alpha = 0.2f)),
        HazeTint(Color.Blue.copy(alpha = 0.1f))
    )
    noiseFactor = 0.15f          // Texture noise (0f to disable)
}
```

Or create a reusable HazeStyle:

```kotlin
val customStyle = HazeStyle(
    blurRadius = 15.dp,
    tints = listOf(HazeTint(Color.Black.copy(alpha = 0.2f))),
    noiseFactor = 0.1f
)

blurEffect { style = customStyle }
```

### Styling Resolution Order

1. Value set directly in `blurEffect {}`
2. Value from `style` property
3. Value from `LocalHazeStyle` composition local
4. Default value

## Progressive (Gradient) Blurs

Vary blur intensity across a dimension:

```kotlin
blurEffect {
    progressive = HazeProgressive.verticalGradient(
        startIntensity = 1f,  // Full blur at top
        endIntensity = 0f     // No blur at bottom
    )
}
```

### Progressive Types

```kotlin
// Vertical gradient
progressive = HazeProgressive.verticalGradient(startIntensity = 1f, endIntensity = 0f)

// Horizontal gradient
progressive = HazeProgressive.horizontalGradient(startIntensity = 1f, endIntensity = 0f)

// Radial gradient
progressive = HazeProgressive.RadialGradient()

// Custom brush
progressive = HazeProgressive.Brush(brush = Brush.verticalGradient(...))
```

## Masking

Apply any Brush as an opacity mask (faster than progressive blur):

```kotlin
blurEffect {
    mask = Brush.verticalGradient(
        colors = listOf(Color.Black, Color.Transparent)
    )
}
```

## Dynamic Styling

Update blur based on scroll state:

```kotlin
val listState = rememberLazyListState()

TopAppBar(
    modifier = Modifier.hazeEffect(state = hazeState) {
        blurEffect {
            alpha = if (listState.firstVisibleItemIndex == 0) {
                val firstItem = listState.layoutInfo.visibleItemsInfo.firstOrNull()
                firstItem?.let { (it.offset / it.size.height.toFloat()).absoluteValue } ?: 1f
            } else {
                1f
            }
        }
    }
)
```

## Scaffold Pattern

Blur both top and bottom bars:

```kotlin
val hazeState = rememberHazeState()

Scaffold(
    topBar = {
        TopAppBar(
            colors = TopAppBarDefaults.topAppBarColors(Color.Transparent),
            modifier = Modifier.hazeEffect(state = hazeState) {
                blurEffect { style = HazeMaterials.thin() }
            }
        )
    },
    bottomBar = {
        NavigationBar(
            containerColor = Color.Transparent,
            modifier = Modifier.hazeEffect(state = hazeState) {
                blurEffect { style = HazeMaterials.thin() }
            }
        )
    }
) { padding ->
    LazyColumn(
        contentPadding = padding,
        modifier = Modifier.hazeSource(state = hazeState)
    ) {
        // content
    }
}
```

## Sticky Headers

For LazyColumn sticky headers, use hazeSource on items (not the column):

```kotlin
val hazeState = rememberHazeState()

LazyColumn {
    stickyHeader {
        Header(
            modifier = Modifier.hazeEffect(state = hazeState) {
                blurEffect { style = HazeMaterials.thin() }
            }
        )
    }

    items(list) { item ->
        ItemRow(
            modifier = Modifier.hazeSource(hazeState)
        )
    }
}
```

## Overlapping Effects

Multiple cards that both draw and serve as blur sources:

```kotlin
val hazeState = rememberHazeState()

Box {
    Background(modifier = Modifier.hazeSource(hazeState, zIndex = 0f))

    // Rear card - draws background
    Card(
        modifier = Modifier
            .hazeSource(hazeState, zIndex = 1f)
            .hazeEffect(hazeState) { blurEffect { } }
    )

    // Middle card - draws background + rear card
    Card(
        modifier = Modifier
            .hazeSource(hazeState, zIndex = 2f)
            .hazeEffect(hazeState) { blurEffect { } }
    )

    // Front card - draws all previous layers
    Card(
        modifier = Modifier
            .hazeSource(hazeState, zIndex = 3f)
            .hazeEffect(hazeState) { blurEffect { } }
    )
}
```

### Filtering Layers

Control which layers are included:

```kotlin
Modifier
    .hazeSource(hazeState, zIndex = 2f, key = "card-2")
    .hazeEffect(hazeState) {
        canDrawArea = { area -> area.key != "card-2" }  // Exclude self
        blurEffect { }
    }
```

## Dialogs

Blur dialog backgrounds over content:

```kotlin
val hazeState = rememberHazeState()
var showDialog by remember { mutableStateOf(false) }

Box {
    LazyColumn(modifier = Modifier.hazeSource(state = hazeState)) { }

    if (showDialog) {
        Dialog(onDismissRequest = { showDialog = false }) {
            Surface(
                color = MaterialTheme.colorScheme.surface.copy(alpha = 0.2f),
                modifier = Modifier.hazeEffect(state = hazeState) {
                    blurEffect { style = HazeMaterials.regular() }
                }
            ) {
                // Dialog content
            }
        }
    }
}
```

## Input Scale (Performance)

Render blur at lower resolution for better performance:

```kotlin
Modifier.hazeEffect(state = hazeState) {
    inputScale = HazeInputScale.Auto  // Platform defaults
    // or
    inputScale = HazeInputScale.Fixed(0.5f)  // 50% resolution

    blurEffect { }
}
```

Values:
- `HazeInputScale.None` - No scaling (default, highest quality)
- `HazeInputScale.Auto` - Automatic platform-optimized scaling
- `HazeInputScale.Fixed(0.66f)` - Custom scale (0.0-1.0)

## Deep Hierarchies

Use CompositionLocal to avoid passing HazeState through many levels:

```kotlin
val LocalHazeState = compositionLocalOf { HazeState() }

@Composable
fun App() {
    val hazeState = rememberHazeState()
    CompositionLocalProvider(LocalHazeState provides hazeState) {
        MainContent()
    }
}

@Composable
fun DeepChild() {
    Box(modifier = Modifier.hazeEffect(state = LocalHazeState.current) {
        blurEffect { }
    })
}
```

## Enabling/Disabling Blur

Conditionally enable blur:

```kotlin
blurEffect {
    blurEnabled = isBlurSupported && userPrefersBlur
}
```

## HazeEffectScope Properties

Common properties (outside `blurEffect {}`):

| Property | Description |
|----------|-------------|
| `inputScale` | Performance optimization for resolution |
| `drawContentBehind` | Draw source content before effect (default: true) |
| `canDrawArea` | Filter which source layers to include |
| `clipToAreasBounds` | Clip effect to area bounds |

## BlurVisualEffect Properties

Properties inside `blurEffect {}`:

| Property | Description |
|----------|-------------|
| `blurRadius` | Blur strength (default: 20.dp) |
| `tints` | List of color overlays |
| `noiseFactor` | Visual texture (0-1, default: 0.15) |
| `progressive` | Gradient blur |
| `mask` | Opacity mask brush |
| `style` | Pre-built HazeStyle |
| `alpha` | Overall effect opacity |
| `blurEnabled` | Toggle blur on/off |
| `backgroundColor` | Background for tint calculation |
| `fallbackTint` | Tint when blur unavailable |

## References

- [Haze Documentation](https://chrisbanes.github.io/haze)
- [Sample Code](https://github.com/chrisbanes/haze/tree/main/sample)
