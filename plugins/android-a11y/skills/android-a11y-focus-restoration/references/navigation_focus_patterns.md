# Navigation Focus Patterns

Focus restoration patterns for different navigation architectures.

## Jetpack Navigation (Compose)

### Basic NavHost Focus Handling

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "list") {
        composable("list") {
            ListScreen(
                onItemClick = { itemId ->
                    navController.navigate("detail/$itemId")
                }
            )
        }
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) { backStackEntry ->
            DetailScreen(
                itemId = backStackEntry.arguments?.getString("itemId") ?: return@composable,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

### SavedStateHandle for Focus State

```kotlin
@HiltViewModel
class ListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Preserved across process death
    val focusedItemId: StateFlow<String?> =
        savedStateHandle.getStateFlow("focusedItemId", null)

    fun saveFocusedItem(itemId: String) {
        savedStateHandle["focusedItemId"] = itemId
    }
}

@Composable
fun ListScreen(
    onItemClick: (String) -> Unit,
    viewModel: ListViewModel = hiltViewModel()
) {
    val items by viewModel.items.collectAsState()
    val focusedItemId by viewModel.focusedItemId.collectAsState()

    val listState = rememberLazyListState()
    val focusRequesters = remember(items) {
        items.associate { it.id to FocusRequester() }
    }

    // Restore focus when returning to screen
    LaunchedEffect(focusedItemId, items) {
        focusedItemId?.let { id ->
            val index = items.indexOfFirst { it.id == id }
            if (index >= 0) {
                listState.scrollToItem(index)
                delay(100)
                focusRequesters[id]?.requestFocus()
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
                            viewModel.saveFocusedItem(item.id)
                        }
                    }
                    .clickable { onItemClick(item.id) }
            )
        }
    }
}
```

### Back Stack Entry Lifecycle

```kotlin
@Composable
fun ListScreenWithLifecycle(
    navController: NavController,
    viewModel: ListViewModel = hiltViewModel()
) {
    val navBackStackEntry by navController.currentBackStackEntryAsState()

    // Restore focus only when this screen becomes active
    LaunchedEffect(navBackStackEntry) {
        if (navBackStackEntry?.destination?.route == "list") {
            // Screen is now visible, restore focus
            viewModel.focusedItemId.value?.let { id ->
                restoreFocusToItem(id)
            }
        }
    }
}
```

## Fragment Navigation

### Fragment with Focus Restoration

```kotlin
class ListFragment : Fragment() {

    private val viewModel: ListViewModel by viewModels()
    private var adapter: ItemAdapter? = null

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        setupRecyclerView()
        observeFocusState()
    }

    private fun observeFocusState() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                // Only restore when fragment is resumed (visible)
                viewModel.focusedItemId.collect { itemId ->
                    itemId?.let { restoreFocusToItem(it) }
                }
            }
        }
    }

    private fun restoreFocusToItem(itemId: String) {
        val position = adapter?.getPositionForId(itemId) ?: return
        if (position < 0) return

        binding.recyclerView.apply {
            scrollToPosition(position)
            post {
                findViewHolderForAdapterPosition(position)
                    ?.itemView
                    ?.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
            }
        }
    }

    override fun onPause() {
        super.onPause()
        // Focus is saved via adapter's focus tracking
    }
}
```

### Navigation Component with Fragments

```kotlin
// In Navigation graph XML
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:startDestination="@id/listFragment">

    <fragment
        android:id="@+id/listFragment"
        android:name="com.example.ListFragment">
        <action
            android:id="@+id/action_to_detail"
            app:destination="@id/detailFragment"
            app:popUpTo="@id/listFragment"
            app:popUpToSaveState="true" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <argument
            android:name="itemId"
            app:argType="string" />
    </fragment>
</navigation>
```

```kotlin
// ListFragment navigation
fun navigateToDetail(itemId: String) {
    // Save current focus before navigating
    saveFocusState()

    val action = ListFragmentDirections.actionToDetail(itemId)
    findNavController().navigate(action)
}
```

## Activity-Based Navigation

### Single Activity with State Preservation

```kotlin
class MainActivity : AppCompatActivity() {

    private val focusStateManager = FocusStateManager()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Restore focus state
        savedInstanceState?.let {
            focusStateManager.restore(it)
        }
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        focusStateManager.save(outState)
    }

    // Called when returning from another activity
    override fun onResume() {
        super.onResume()
        focusStateManager.restoreFocusDelayed()
    }
}

class FocusStateManager {
    private var focusedViewId: Int? = null
    private var focusedItemPosition: Int? = null

    fun save(outState: Bundle) {
        focusedViewId?.let { outState.putInt(KEY_VIEW_ID, it) }
        focusedItemPosition?.let { outState.putInt(KEY_POSITION, it) }
    }

    fun restore(savedState: Bundle) {
        focusedViewId = savedState.getInt(KEY_VIEW_ID, -1).takeIf { it != -1 }
        focusedItemPosition = savedState.getInt(KEY_POSITION, -1).takeIf { it != -1 }
    }

    fun restoreFocusDelayed() {
        // Post to allow views to be ready
        handler.post {
            focusedViewId?.let { viewId ->
                activity.findViewById<View>(viewId)?.let { view ->
                    if (view is RecyclerView) {
                        focusedItemPosition?.let { position ->
                            restoreRecyclerViewFocus(view, position)
                        }
                    } else {
                        view.sendAccessibilityEvent(
                            AccessibilityEvent.TYPE_VIEW_FOCUSED
                        )
                    }
                }
            }
        }
    }

    private fun restoreRecyclerViewFocus(rv: RecyclerView, position: Int) {
        rv.scrollToPosition(position)
        rv.post {
            rv.findViewHolderForAdapterPosition(position)
                ?.itemView
                ?.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
        }
    }

    companion object {
        private const val KEY_VIEW_ID = "focus_view_id"
        private const val KEY_POSITION = "focus_position"
    }
}
```

## Deep Link Focus Handling

### Compose Navigation with Deep Links

```kotlin
NavHost(navController, startDestination = "home") {
    composable(
        route = "item/{itemId}",
        deepLinks = listOf(navDeepLink { uriPattern = "app://item/{itemId}" })
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId")

        // Focus the item when arriving via deep link
        ItemDetailScreen(
            itemId = itemId,
            shouldAutoFocus = backStackEntry.destination.route?.contains("deepLink") == true
        )
    }
}

@Composable
fun ItemDetailScreen(
    itemId: String?,
    shouldAutoFocus: Boolean = false
) {
    val focusRequester = remember { FocusRequester() }

    LaunchedEffect(shouldAutoFocus) {
        if (shouldAutoFocus) {
            focusRequester.requestFocus()
        }
    }

    Column {
        Text(
            text = "Item: $itemId",
            modifier = Modifier
                .focusRequester(focusRequester)
                .semantics { heading() }
        )
        // Content
    }
}
```

## Tab Navigation Focus

### Bottom Navigation with Focus Restoration

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    // Track focus per tab
    val focusStates = remember { mutableStateMapOf<String, String?>() }

    Scaffold(
        bottomBar = {
            BottomNavigation {
                tabs.forEach { tab ->
                    BottomNavigationItem(
                        selected = currentRoute == tab.route,
                        onClick = {
                            // Save current focus before switching
                            focusStates[currentRoute] = getCurrentFocusedId()

                            navController.navigate(tab.route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(tab.icon, contentDescription = tab.label) },
                        label = { Text(tab.label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") {
                HomeScreen(
                    savedFocusId = focusStates["home"],
                    onFocusChanged = { focusStates["home"] = it }
                )
            }
            composable("search") {
                SearchScreen(
                    savedFocusId = focusStates["search"],
                    onFocusChanged = { focusStates["search"] = it }
                )
            }
            composable("profile") {
                ProfileScreen(
                    savedFocusId = focusStates["profile"],
                    onFocusChanged = { focusStates["profile"] = it }
                )
            }
        }
    }
}
```

## Testing Navigation Focus

```kotlin
@Test
fun focusRestoredAfterNavigationBack() = runTest {
    composeTestRule.setContent {
        AppNavigation()
    }

    // Navigate to list and focus item
    composeTestRule.onNodeWithText("Item 3").performClick()
    composeTestRule.onNodeWithText("Item 3").assertIsFocused()

    // Navigate to detail
    composeTestRule.onNodeWithText("View Details").performClick()
    composeTestRule.onNodeWithText("Item 3 Details").assertExists()

    // Navigate back
    composeTestRule.onNodeWithContentDescription("Back").performClick()

    // Verify focus restored
    composeTestRule.waitForIdle()
    composeTestRule.onNodeWithText("Item 3").assertIsFocused()
}
```
