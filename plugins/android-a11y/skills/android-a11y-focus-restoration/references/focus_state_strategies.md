# Focus State Strategies

Comparison of different approaches for saving and restoring accessibility focus.

## Strategy Comparison

| Strategy | Survives Config Change | Survives Process Death | Best For |
|----------|----------------------|----------------------|----------|
| `remember` | No | No | Temporary state |
| `rememberSaveable` | Yes | Yes | Most cases |
| `ViewModel` | Yes | No | Complex state |
| `SavedStateHandle` | Yes | Yes | Navigation args |
| `onSaveInstanceState` | Yes | Yes | Fragment/Activity |

## Position-Based Restoration

### When to Use
- Ordered lists that don't change
- Pagination where position is stable
- Simple scroll restoration

### Implementation

```kotlin
@Composable
fun PositionBasedFocusRestoration(items: List<Item>) {
    // Save position index
    var focusedIndex by rememberSaveable { mutableStateOf<Int?>(null) }

    val listState = rememberLazyListState()

    // Restore on recomposition
    LaunchedEffect(focusedIndex) {
        focusedIndex?.let { index ->
            if (index < items.size) {
                listState.scrollToItem(index)
            }
        }
    }

    LazyColumn(state = listState) {
        itemsIndexed(items) { index, item ->
            ItemRow(
                item = item,
                modifier = Modifier
                    .focusable()
                    .onFocusChanged { state ->
                        if (state.isFocused) {
                            focusedIndex = index
                        }
                    }
            )
        }
    }
}
```

### Pros
- Simple to implement
- Works with any list
- Efficient (no search needed)

### Cons
- Breaks if items inserted/removed before focused position
- Can focus wrong item after list mutation

## ID-Based Restoration

### When to Use
- Lists with stable item IDs
- Data that may be reordered
- Most production apps

### Implementation

```kotlin
@Composable
fun IdBasedFocusRestoration(items: List<Item>) {
    // Save item ID
    var focusedId by rememberSaveable { mutableStateOf<String?>(null) }

    val listState = rememberLazyListState()
    val focusRequesters = remember(items) {
        items.associate { it.id to FocusRequester() }
    }

    // Find and restore
    LaunchedEffect(focusedId, items) {
        focusedId?.let { id ->
            val index = items.indexOfFirst { it.id == id }
            if (index >= 0) {
                listState.scrollToItem(index)
                delay(100)
                focusRequesters[id]?.requestFocus()
            } else {
                // Item was removed, clear saved ID
                focusedId = null
            }
        }
    }

    LazyColumn(state = listState) {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                modifier = Modifier
                    .focusRequester(focusRequesters[item.id]!!)
                    .onFocusChanged { state ->
                        if (state.isFocused) {
                            focusedId = item.id
                        }
                    }
            )
        }
    }
}
```

### Pros
- Survives reordering
- Handles insertions/deletions gracefully
- Focuses correct item even after data changes

### Cons
- Requires stable IDs
- Linear search to find index (O(n))
- More complex implementation

## Content-Based Restoration

### When to Use
- No stable IDs available
- Unique, identifying content exists
- Legacy data sources

### Implementation

```kotlin
@Composable
fun ContentBasedFocusRestoration(items: List<LegacyItem>) {
    // Save identifying content
    var focusedTitle by rememberSaveable { mutableStateOf<String?>(null) }

    val listState = rememberLazyListState()

    LaunchedEffect(focusedTitle, items) {
        focusedTitle?.let { title ->
            val index = items.indexOfFirst { it.title == title }
            if (index >= 0) {
                listState.scrollToItem(index)
            }
        }
    }

    LazyColumn(state = listState) {
        itemsIndexed(items) { index, item ->
            ItemRow(
                item = item,
                modifier = Modifier
                    .focusable()
                    .onFocusChanged { state ->
                        if (state.isFocused) {
                            focusedTitle = item.title
                        }
                    }
            )
        }
    }
}
```

### Pros
- Works without IDs
- Can match across data refreshes

### Cons
- Fragile if content changes
- Duplicate content causes wrong match
- Slower than ID-based

## Hybrid Strategy

Combine strategies for robustness:

```kotlin
@Composable
fun HybridFocusRestoration(items: List<Item>) {
    // Save multiple identifiers
    var focusState by rememberSaveable {
        mutableStateOf(FocusState(null, null, null))
    }

    val listState = rememberLazyListState()

    LaunchedEffect(focusState, items) {
        val state = focusState

        // Try ID first
        state.id?.let { id ->
            val index = items.indexOfFirst { it.id == id }
            if (index >= 0) {
                listState.scrollToItem(index)
                return@LaunchedEffect
            }
        }

        // Fall back to content match
        state.title?.let { title ->
            val index = items.indexOfFirst { it.title == title }
            if (index >= 0) {
                listState.scrollToItem(index)
                return@LaunchedEffect
            }
        }

        // Last resort: position (may be wrong item)
        state.position?.let { position ->
            if (position < items.size) {
                listState.scrollToItem(position)
            }
        }
    }

    LazyColumn(state = listState) {
        itemsIndexed(items, key = { _, item -> item.id }) { index, item ->
            ItemRow(
                item = item,
                modifier = Modifier
                    .focusable()
                    .onFocusChanged { state ->
                        if (state.isFocused) {
                            focusState = FocusState(
                                id = item.id,
                                title = item.title,
                                position = index
                            )
                        }
                    }
            )
        }
    }
}

@Parcelize
data class FocusState(
    val id: String?,
    val title: String?,
    val position: Int?
) : Parcelable
```

## ViewModel Integration

For complex apps with shared state:

```kotlin
@HiltViewModel
class ListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Survives process death via SavedStateHandle
    private val _focusedItemId = savedStateHandle.getStateFlow<String?>(
        KEY_FOCUSED_ID, null
    )
    val focusedItemId = _focusedItemId

    // Additional state for complex scenarios
    private val _scrollPosition = savedStateHandle.getStateFlow(
        KEY_SCROLL_POSITION, 0
    )

    fun onItemFocused(itemId: String, position: Int) {
        savedStateHandle[KEY_FOCUSED_ID] = itemId
        savedStateHandle[KEY_SCROLL_POSITION] = position
    }

    fun clearFocus() {
        savedStateHandle[KEY_FOCUSED_ID] = null
    }

    companion object {
        private const val KEY_FOCUSED_ID = "focused_item_id"
        private const val KEY_SCROLL_POSITION = "scroll_position"
    }
}

@Composable
fun ListScreen(viewModel: ListViewModel = hiltViewModel()) {
    val focusedId by viewModel.focusedItemId.collectAsState()

    // Use focusedId for restoration...
}
```

## Testing Focus Restoration

```kotlin
@Test
fun focusRestoredAfterConfigurationChange() {
    val scenario = launchActivity<MainActivity>()

    // Focus an item
    composeTestRule
        .onNodeWithText("Item 5")
        .performClick()

    // Verify focused
    composeTestRule
        .onNodeWithText("Item 5")
        .assertIsFocused()

    // Simulate configuration change
    scenario.recreate()

    // Verify focus restored
    composeTestRule
        .onNodeWithText("Item 5")
        .assertIsFocused()
}

@Test
fun focusRestoredAfterNavigation() {
    // Navigate to detail
    composeTestRule
        .onNodeWithText("Item 5")
        .performClick()

    // Verify on detail screen
    composeTestRule
        .onNodeWithText("Item 5 Details")
        .assertExists()

    // Navigate back
    composeTestRule.activityRule.scenario.onActivity {
        it.onBackPressedDispatcher.onBackPressed()
    }

    // Verify focus restored to Item 5
    composeTestRule
        .onNodeWithText("Item 5")
        .assertIsFocused()
}
```
