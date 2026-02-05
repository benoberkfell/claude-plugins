---
name: android-a11y-focus-restoration
description: Preserve and restore accessibility focus during navigation, configuration changes, and screen transitions. Use when focus is lost after navigating back, rotating device, or when list position resets.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Focus Preservation and Restoration

This skill covers how to maintain accessibility focus continuity across navigation events, configuration changes, and screen transitions.

## Why Focus Gets Lost

Focus is lost when the accessibility node tree changes:

1. **Navigation** - Fragment/screen replaced, old nodes destroyed
2. **Configuration change** - Activity recreated, all nodes rebuilt
3. **List scroll** - Items recycled, node IDs change
4. **Recomposition** - Compose nodes rebuilt with new identities

## Fragment Focus Preservation

### The Problem

```kotlin
// User navigates: ListFragment → DetailFragment → Back
// Focus was on "Item 5" in ListFragment
// After back navigation, focus resets to top of list
```

### Solution: Save Focus State in ViewModel

```kotlin
class ListViewModel : ViewModel() {
    // Track which item had focus
    var focusedItemId: String? by mutableStateOf(null)
        private set

    fun saveFocusedItem(itemId: String) {
        focusedItemId = itemId
    }

    fun clearFocusedItem() {
        focusedItemId = null
    }
}

class ListFragment : Fragment() {
    private val viewModel: ListViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Restore focus after view is ready
        view.post {
            viewModel.focusedItemId?.let { itemId ->
                restoreFocusToItem(itemId)
            }
        }
    }

    private fun restoreFocusToItem(itemId: String) {
        val position = adapter.getPositionForId(itemId)
        if (position >= 0) {
            recyclerView.scrollToPosition(position)
            recyclerView.post {
                recyclerView.findViewHolderForAdapterPosition(position)
                    ?.itemView
                    ?.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
            }
        }
    }

    // Save focus when item is interacted with
    private fun onItemFocused(itemId: String) {
        viewModel.saveFocusedItem(itemId)
    }
}
```

### Track Focus via AccessibilityDelegate

```kotlin
class FocusTrackingViewHolder(
    itemView: View,
    private val onFocused: (String) -> Unit
) : RecyclerView.ViewHolder(itemView) {

    private var currentItemId: String? = null

    init {
        itemView.accessibilityDelegate = object : View.AccessibilityDelegate() {
            override fun sendAccessibilityEvent(host: View, eventType: Int) {
                super.sendAccessibilityEvent(host, eventType)

                if (eventType == AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED) {
                    currentItemId?.let { onFocused(it) }
                }
            }
        }
    }

    fun bind(item: Item) {
        currentItemId = item.id
        // ... bind other data
    }
}
```

## Compose Navigation Focus Restoration

### Basic Pattern with rememberSaveable

```kotlin
@Composable
fun ItemListScreen(
    items: List<Item>,
    onItemClick: (Item) -> Unit,
    viewModel: ListViewModel = viewModel()
) {
    // Survive process death
    var focusedItemId by rememberSaveable { mutableStateOf<String?>(null) }

    // FocusRequesters for each item
    val focusRequesters = remember(items) {
        items.associate { it.id to FocusRequester() }
    }

    // Restore focus on composition
    LaunchedEffect(focusedItemId) {
        focusedItemId?.let { id ->
            focusRequesters[id]?.requestFocus()
        }
    }

    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                modifier = Modifier
                    .focusRequester(focusRequesters[item.id]!!)
                    .onFocusChanged { focusState ->
                        if (focusState.isFocused) {
                            focusedItemId = item.id
                        }
                    }
                    .clickable { onItemClick(item) }
            )
        }
    }
}
```

### With FocusRestorer (Compose 1.6+)

```kotlin
@OptIn(ExperimentalComposeUiApi::class)
@Composable
fun ItemListWithFocusRestorer(
    items: List<Item>,
    modifier: Modifier = Modifier
) {
    val focusManager = LocalFocusManager.current

    LazyColumn(
        modifier = modifier.focusRestorer()  // Automatically restores focus
    ) {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                modifier = Modifier.focusable()
            )
        }
    }
}
```

### Navigation with SavedStateHandle

```kotlin
@Composable
fun ListScreen(
    navController: NavController,
    viewModel: ListViewModel = hiltViewModel()
) {
    // Restore from SavedStateHandle (survives process death)
    val focusedId by viewModel.focusedItemId.collectAsState()

    val listState = rememberLazyListState()
    val focusRequesters = remember { mutableMapOf<String, FocusRequester>() }

    // Restore scroll position and focus
    LaunchedEffect(focusedId) {
        focusedId?.let { id ->
            val index = viewModel.getIndexForId(id)
            if (index >= 0) {
                listState.scrollToItem(index)
                delay(100)  // Wait for scroll
                focusRequesters[id]?.requestFocus()
            }
        }
    }

    LazyColumn(state = listState) {
        items(viewModel.items, key = { it.id }) { item ->
            val focusRequester = remember { FocusRequester() }
                .also { focusRequesters[item.id] = it }

            ItemRow(
                item = item,
                modifier = Modifier
                    .focusRequester(focusRequester)
                    .onFocusChanged {
                        if (it.isFocused) {
                            viewModel.setFocusedItem(item.id)
                        }
                    }
                    .clickable {
                        navController.navigate("detail/${item.id}")
                    }
            )
        }
    }
}

@HiltViewModel
class ListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    val focusedItemId = savedStateHandle.getStateFlow("focusedId", null as String?)

    fun setFocusedItem(id: String) {
        savedStateHandle["focusedId"] = id
    }
}
```

## Initial Focus on Screen Entry

### Compose: Focus First Interactive Element

```kotlin
@Composable
fun LoginScreen() {
    val usernameFocusRequester = remember { FocusRequester() }

    // Request focus after composition
    LaunchedEffect(Unit) {
        usernameFocusRequester.requestFocus()
    }

    Column {
        Text(
            text = "Sign In",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics { heading() }
        )

        TextField(
            value = username,
            onValueChange = { username = it },
            label = { Text("Username") },
            modifier = Modifier.focusRequester(usernameFocusRequester)
        )

        TextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") }
        )

        Button(onClick = { login() }) {
            Text("Sign In")
        }
    }
}
```

### Compose: Announce Screen Title

```kotlin
@Composable
fun ScreenWithAnnouncement(title: String, content: @Composable () -> Unit) {
    val view = LocalView.current

    // Announce screen title for TalkBack
    LaunchedEffect(title) {
        view.announceForAccessibility(title)
    }

    Column {
        Text(
            text = title,
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics { heading() }
        )
        content()
    }
}
```

### XML: requestFocus

```xml
<EditText
    android:id="@+id/usernameInput"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:hint="Username">
    <requestFocus />
</EditText>
```

## Configuration Change Handling

### ViewModel + rememberSaveable

```kotlin
@Composable
fun RobustFocusHandling(
    viewModel: MyViewModel = viewModel()
) {
    // Survives configuration change AND process death
    var focusedId by rememberSaveable { mutableStateOf(viewModel.lastFocusedId) }

    // Sync back to ViewModel (for other purposes)
    LaunchedEffect(focusedId) {
        viewModel.lastFocusedId = focusedId
    }

    // Restore focus
    val focusRequester = remember { FocusRequester() }
    LaunchedEffect(focusedId) {
        if (focusedId != null) {
            focusRequester.requestFocus()
        }
    }

    // UI
}
```

### Fragment + onSaveInstanceState

```kotlin
class ListFragment : Fragment() {

    private var focusedPosition: Int = RecyclerView.NO_POSITION

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putInt(KEY_FOCUSED_POSITION, focusedPosition)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        focusedPosition = savedInstanceState?.getInt(KEY_FOCUSED_POSITION)
            ?: RecyclerView.NO_POSITION

        if (focusedPosition != RecyclerView.NO_POSITION) {
            view.post { restoreFocus(focusedPosition) }
        }
    }

    private fun restoreFocus(position: Int) {
        recyclerView.scrollToPosition(position)
        recyclerView.post {
            recyclerView.findViewHolderForAdapterPosition(position)
                ?.itemView
                ?.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
        }
    }

    companion object {
        private const val KEY_FOCUSED_POSITION = "focused_position"
    }
}
```

## Focus Restoration Strategies

### Strategy 1: Position-Based

Best for: Ordered lists where position is stable

```kotlin
// Save position
val focusedPosition = layoutManager.findFirstVisibleItemPosition()

// Restore by position
listState.scrollToItem(focusedPosition)
```

### Strategy 2: ID-Based

Best for: Lists where items may be reordered/inserted/removed

```kotlin
// Save item ID
val focusedItemId = adapter.getItem(position).id

// Restore by finding item with ID
val newPosition = items.indexOfFirst { it.id == focusedItemId }
if (newPosition >= 0) {
    listState.scrollToItem(newPosition)
}
```

### Strategy 3: Content-Based

Best for: When IDs aren't stable but content is unique

```kotlin
// Save identifying content
val focusedItemTitle = items[position].title

// Find by content match
val newPosition = items.indexOfFirst { it.title == focusedItemTitle }
```

## Common Pitfalls

### Pitfall 1: Focus Request Before Composition

```kotlin
// Problem: FocusRequester not attached yet
val focusRequester = remember { FocusRequester() }
focusRequester.requestFocus()  // Crashes!

// Solution: Use LaunchedEffect
LaunchedEffect(Unit) {
    focusRequester.requestFocus()
}
```

### Pitfall 2: Focus Lost in LazyColumn

```kotlin
// Problem: No stable keys
LazyColumn {
    items(data) { item ->  // Index-based keys
        FocusableItem(item)
    }
}

// Solution: Stable keys
LazyColumn {
    items(data, key = { it.id }) { item ->
        FocusableItem(item)
    }
}
```

### Pitfall 3: Multiple Focus Requests

```kotlin
// Problem: Racing focus requests
LaunchedEffect(condition1) { focusRequester1.requestFocus() }
LaunchedEffect(condition2) { focusRequester2.requestFocus() }

// Solution: Single source of truth
val focusTarget by remember { derivedStateOf {
    when {
        condition1 -> focusRequester1
        condition2 -> focusRequester2
        else -> null
    }
}}

LaunchedEffect(focusTarget) {
    focusTarget?.requestFocus()
}
```

See `references/focus_state_strategies.md` for detailed strategy comparison.
See `references/navigation_focus_patterns.md` for Navigation Component patterns.
