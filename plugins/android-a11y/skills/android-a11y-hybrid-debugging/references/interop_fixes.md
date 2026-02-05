# View/Compose Interop Accessibility Fixes

Solutions for common accessibility issues in hybrid View/Compose apps.

## ComposeView Integration

### Problem: ComposeView Steals Focus

**Symptom**: Focus jumps unexpectedly when scrolling RecyclerView with ComposeView items.

**Cause**: `AndroidComposeView` is focusable by default, competing with RecyclerView focus management.

**Fix**:

```kotlin
class ComposeViewHolder(
    private val composeView: ComposeView
) : RecyclerView.ViewHolder(composeView) {

    init {
        composeView.apply {
            // Disable ComposeView's native focus handling
            isFocusable = false
            isFocusableInTouchMode = false

            // Let Compose content handle accessibility
            importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS
        }
    }

    fun bind(item: Item) {
        composeView.setContent {
            // Compose semantics work normally
            ItemCard(
                item = item,
                modifier = Modifier.semantics(mergeDescendants = true) {
                    contentDescription = "${item.title}, ${item.subtitle}"
                }
            )
        }
    }
}
```

### Problem: Lifecycle Issues

**Symptom**: Compose content doesn't update accessibility tree after recomposition.

**Fix**:

```kotlin
class ComposeViewHolder(
    private val composeView: ComposeView
) : RecyclerView.ViewHolder(composeView) {

    init {
        // Ensure proper lifecycle handling
        composeView.setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
        )
    }
}
```

## AndroidView Integration

### Problem: Double Focus

**Symptom**: Both Compose and the inner View can receive focus, causing duplicate announcements.

**Fix**:

```kotlin
@Composable
fun LegacyButtonWrapper(
    label: String,
    onClick: () -> Unit
) {
    AndroidView(
        factory = { context ->
            Button(context).apply {
                text = label
                setOnClickListener { onClick() }
                // View handles its own accessibility
            }
        },
        // Don't add competing focusable modifier
        modifier = Modifier
            // .focusable()  // Don't do this!
    )
}
```

### Problem: Conflicting Semantics

**Symptom**: Compose semantics override or conflict with View's AccessibilityNodeInfo.

**Fix**: Choose one layer to handle accessibility:

```kotlin
// Option 1: Let View handle it
@Composable
fun ViewHandlesA11y(item: Item) {
    AndroidView(
        factory = { context ->
            CustomView(context).apply {
                // View configures its own accessibility
                contentDescription = item.accessibleDescription
            }
        }
        // No semantics modifier - View handles it
    )
}

// Option 2: Compose overrides View
@Composable
fun ComposeHandlesA11y(item: Item) {
    AndroidView(
        factory = { context ->
            CustomView(context).apply {
                // Disable View accessibility
                importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO
            }
        },
        modifier = Modifier.semantics {
            // Compose provides all accessibility info
            contentDescription = item.accessibleDescription
            role = Role.Button
            onClick { /* ... */ }
        }
    )
}
```

## WebView Integration

### Problem: WebView Accessibility Tree Not Visible

**Symptom**: TalkBack skips over WebView content.

**Fix**:

```kotlin
@Composable
fun AccessibleWebView(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                // Required settings
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true

                // Accessibility settings
                importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

                // Optional: Custom accessibility delegate
                accessibilityDelegate = WebViewAccessibilityDelegate()
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        },
        modifier = Modifier.semantics {
            isTraversalGroup = true
            contentDescription = "Web content"
        }
    )
}

class WebViewAccessibilityDelegate : View.AccessibilityDelegate() {
    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfo) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        // Hint for users
        info.hintText = "Web page. Explore mode available."
    }
}
```

### Problem: WebView Focus Traps

**Symptom**: Can't navigate out of WebView with TalkBack.

**Fix**:

```kotlin
@Composable
fun WebViewWithEscape(url: String) {
    var showEscapeHint by remember { mutableStateOf(false) }

    Column {
        // Escape hint above WebView
        if (showEscapeHint) {
            Text(
                "Swipe up then right to exit web content",
                modifier = Modifier.semantics {
                    liveRegion = LiveRegionMode.Polite
                }
            )
        }

        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

                    // Track when WebView has focus
                    accessibilityDelegate = object : View.AccessibilityDelegate() {
                        override fun sendAccessibilityEvent(host: View, eventType: Int) {
                            if (eventType == AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED) {
                                showEscapeHint = true
                            } else if (eventType == AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUS_CLEARED) {
                                showEscapeHint = false
                            }
                            super.sendAccessibilityEvent(host, eventType)
                        }
                    }
                }
            },
            update = { it.loadUrl(url) }
        )
    }
}
```

## Dialog Interop

### Problem: Dialog Focus Doesn't Trap

**Symptom**: TalkBack navigates behind AlertDialog to content below.

**Fix for Compose Dialog in View Activity**:

```kotlin
// In Activity
fun showComposeDialog() {
    val dialog = Dialog(this).apply {
        setContentView(ComposeView(context).apply {
            setContent {
                DialogContent(onDismiss = { dismiss() })
            }
        })

        window?.apply {
            // Ensure proper window type for accessibility
            setType(WindowManager.LayoutParams.TYPE_APPLICATION_PANEL)

            // Set proper focus behavior
            decorView.importantForAccessibility =
                View.IMPORTANT_FOR_ACCESSIBILITY_YES
        }
    }
    dialog.show()
}

@Composable
fun DialogContent(onDismiss: () -> Unit) {
    Surface(
        modifier = Modifier.semantics {
            // Announce as dialog
            paneTitle = "Alert"
        }
    ) {
        Column {
            Text("Dialog Title", Modifier.semantics { heading() })
            Text("Dialog content")
            Button(onClick = onDismiss) { Text("Close") }
        }
    }
}
```

## Bottom Sheet Interop

### Problem: Bottom Sheet Focus Order

**Symptom**: TalkBack reads bottom sheet before content above it.

**Fix**:

```kotlin
@Composable
fun ScreenWithBottomSheet() {
    Box {
        // Main content
        MainContent(
            modifier = Modifier.semantics {
                isTraversalGroup = true
                traversalIndex = 0f
            }
        )

        // Bottom sheet - read after main content
        BottomSheet(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .semantics {
                    isTraversalGroup = true
                    traversalIndex = 1f
                    paneTitle = "Options panel"
                }
        )
    }
}
```

## Fragment + Compose Interop

### Problem: Focus Lost During Fragment Transaction

**Fix**:

```kotlin
class HybridFragment : Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        val composeView = view.findViewById<ComposeView>(R.id.compose_container)

        composeView.setContent {
            val lifecycleOwner = LocalLifecycleOwner.current

            // Restore focus when fragment resumes
            LaunchedEffect(lifecycleOwner) {
                lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                    restoreFocus()
                }
            }

            ScreenContent()
        }
    }
}
```

## RecyclerView Header/Footer with Compose

### Problem: Compose Headers Break RecyclerView Accessibility

**Fix**:

```kotlin
class HeaderAdapter(
    private val headerContent: @Composable () -> Unit
) : RecyclerView.Adapter<HeaderAdapter.HeaderViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): HeaderViewHolder {
        return HeaderViewHolder(ComposeView(parent.context).apply {
            // Same fixes as regular ComposeViewHolder
            isFocusable = false
            isFocusableInTouchMode = false
            layoutParams = RecyclerView.LayoutParams(
                RecyclerView.LayoutParams.MATCH_PARENT,
                RecyclerView.LayoutParams.WRAP_CONTENT
            )
        })
    }

    override fun onBindViewHolder(holder: HeaderViewHolder, position: Int) {
        holder.bind(headerContent)
    }

    override fun getItemCount() = 1

    class HeaderViewHolder(
        private val composeView: ComposeView
    ) : RecyclerView.ViewHolder(composeView) {

        fun bind(content: @Composable () -> Unit) {
            composeView.setContent {
                Box(
                    modifier = Modifier.semantics {
                        // Mark as heading for easier navigation
                        heading()
                    }
                ) {
                    content()
                }
            }
        }
    }
}

// Use with ConcatAdapter
val concatAdapter = ConcatAdapter(
    headerAdapter,
    mainItemsAdapter,
    footerAdapter
)
recyclerView.adapter = concatAdapter
```

## Testing Interop Accessibility

```kotlin
@Test
fun composeInRecyclerView_hasCorrectA11y() {
    // Set up RecyclerView with ComposeView items
    val items = listOf(Item("1", "Title 1"), Item("2", "Title 2"))

    activityRule.scenario.onActivity { activity ->
        val recyclerView = activity.findViewById<RecyclerView>(R.id.recyclerView)
        recyclerView.adapter = HybridAdapter(items)
    }

    // Verify first item is accessible
    onView(withContentDescription(containsString("Title 1")))
        .check(matches(isDisplayed()))

    // Verify focus can move between items
    onView(withContentDescription(containsString("Title 1")))
        .perform(pressKey(KeyEvent.KEYCODE_TAB))

    onView(withContentDescription(containsString("Title 2")))
        .check(matches(hasFocus()))
}
```
