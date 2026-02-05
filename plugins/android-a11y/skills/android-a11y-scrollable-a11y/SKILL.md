---
name: android-a11y-scrollable-a11y
description: Fix accessibility issues in scrollable containers like RecyclerView, LazyColumn, HorizontalPager, and carousels. Use when TalkBack can't scroll, focus gets trapped, or items are skipped during navigation.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Scrollable Container Accessibility

This skill covers accessibility issues specific to scrollable containers - RecyclerView, LazyColumn/LazyRow, HorizontalPager, and carousels.

## RecyclerView Scroll Blocking

### The Problem

TalkBack uses `ACTION_SCROLL_FORWARD` and `ACTION_SCROLL_BACKWARD` to scroll containers. If these actions are missing, TalkBack can't scroll.

**Common cause**: `importantForAccessibility="no"` on RecyclerView or parent.

```xml
<!-- Problem: Strips scroll actions from accessibility tree -->
<RecyclerView
    android:importantForAccessibility="no"
    ... />
```

### The Fix

```xml
<!-- Solution: Ensure RecyclerView is important for accessibility -->
<RecyclerView
    android:importantForAccessibility="yes"
    ... />
```

```kotlin
// Or in code
recyclerView.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES
```

### Verify Scroll Actions Exist

```kotlin
// Debug: Check if scroll actions are exposed
recyclerView.accessibilityDelegate = object : View.AccessibilityDelegate() {
    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfo) {
        super.onInitializeAccessibilityNodeInfo(host, info)
        Log.d("A11y", "Actions: ${info.actionList.map { it.id }}")
        // Should include ACTION_SCROLL_FORWARD (4096) and ACTION_SCROLL_BACKWARD (8192)
    }
}
```

## Focus Traps in Carousels

### The Problem

Horizontal carousels can trap TalkBack focus when both the container and children compete for focus.

```kotlin
// Problem: Container's descendantFocusability conflicts with child focus
<HorizontalScrollView
    android:descendantFocusability="blocksDescendants">  <!-- Blocks child focus -->
    <LinearLayout>
        <CardView android:focusable="true" />  <!-- Can't receive focus! -->
    </LinearLayout>
</HorizontalScrollView>
```

### The Fix

```xml
<!-- Solution: Allow descendants to receive focus after container -->
<HorizontalScrollView
    android:descendantFocusability="afterDescendants">
    <LinearLayout>
        <CardView android:focusable="true" />
    </LinearLayout>
</HorizontalScrollView>
```

```kotlin
// In code
horizontalScrollView.descendantFocusability = ViewGroup.FOCUS_AFTER_DESCENDANTS
```

### descendantFocusability Values

| Value | Behavior | Use Case |
|-------|----------|----------|
| `beforeDescendants` | Container gets focus first | Rarely needed |
| `afterDescendants` | Children get focus first | **Recommended for carousels** |
| `blocksDescendants` | Children never get focus | Only for decorative containers |

## HorizontalPager Focus Jump

### The Problem

In Jetpack Compose's HorizontalPager, swiping pages recreates nodes, causing accessibility focus to jump unexpectedly.

```kotlin
// Problem: Focus lost when page changes
HorizontalPager(state = pagerState) { page ->
    PageContent(data[page])  // New composition = new node = focus lost
}
```

### The Fix

Use `FocusRequester` to restore focus after page change:

```kotlin
@Composable
fun AccessiblePager(pages: List<PageData>) {
    val pagerState = rememberPagerState { pages.size }
    val focusRequesters = remember {
        List(pages.size) { FocusRequester() }
    }

    // Restore focus when page settles
    LaunchedEffect(pagerState.currentPage) {
        // Small delay for composition to complete
        delay(100)
        focusRequesters[pagerState.currentPage].requestFocus()
    }

    HorizontalPager(state = pagerState) { page ->
        Column(
            modifier = Modifier
                .focusRequester(focusRequesters[page])
                .semantics {
                    // Announce page position
                    contentDescription = "Page ${page + 1} of ${pages.size}"
                }
        ) {
            PageContent(pages[page])
        }
    }
}
```

### Alternative: Page Indicator Announcement

```kotlin
// Announce page change via live region
@Composable
fun PagerWithAnnouncement(pages: List<PageData>) {
    val pagerState = rememberPagerState { pages.size }

    Column {
        HorizontalPager(state = pagerState) { page ->
            PageContent(pages[page])
        }

        // Page indicator that announces changes
        Text(
            text = "Page ${pagerState.currentPage + 1} of ${pages.size}",
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}
```

## LazyColumn/LazyRow Focus Issues

### Issue 1: Focus Lost During Scroll

```kotlin
// Problem: Items recompose with different keys
LazyColumn {
    items(data) { item ->  // Implicit index-based key
        ItemRow(item)  // Key changes if list order changes
    }
}

// Solution: Stable keys
LazyColumn {
    items(data, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

### Issue 2: Can't Scroll Past Visible Items

```kotlin
// Problem: LazyColumn inside fixed-height container
Column(Modifier.height(200.dp)) {
    LazyColumn {  // Can't scroll!
        items(100) { Text("Item $it") }
    }
}

// Solution: Give LazyColumn room to scroll
Column {
    LazyColumn(Modifier.weight(1f)) {
        items(100) { Text("Item $it") }
    }
}
```

### Issue 3: Missing Collection Info

TalkBack announces "in list" and item positions when collection info is present:

```kotlin
LazyColumn(
    modifier = Modifier.semantics {
        collectionInfo = CollectionInfo(
            rowCount = items.size,
            columnCount = 1
        )
    }
) {
    itemsIndexed(items) { index, item ->
        Row(
            modifier = Modifier.semantics {
                collectionItemInfo = CollectionItemInfo(
                    rowIndex = index,
                    rowSpan = 1,
                    columnIndex = 0,
                    columnSpan = 1
                )
            }
        ) {
            // Item content
        }
    }
}
```

## ViewPager2 Accessibility

### Basic Setup

```kotlin
viewPager2.apply {
    // Ensure accessibility is enabled
    importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

    // Register page change callback for announcements
    registerOnPageChangeCallback(object : ViewPager2.OnPageChangeCallback() {
        override fun onPageSelected(position: Int) {
            // Announce page change
            announceForAccessibility("Page ${position + 1} of $pageCount")
        }
    })
}
```

### Tab Layout Integration

```kotlin
// TabLayout + ViewPager2 accessibility
TabLayoutMediator(tabLayout, viewPager) { tab, position ->
    tab.text = "Tab ${position + 1}"
    tab.contentDescription = "Tab ${position + 1} of ${adapter.itemCount}"
}.attach()
```

## ComposeView in RecyclerView

### The Problem

When using `ComposeView` inside RecyclerView items, the AndroidComposeView can steal accessibility focus from sibling items.

```kotlin
// Problem: Focus jumps between Compose and View items unpredictably
class HybridAdapter : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        when (viewType) {
            TYPE_COMPOSE -> ComposeViewHolder(ComposeView(parent.context))
            else -> ViewViewHolder(LayoutInflater.inflate(...))
        }
}
```

### The Fix

Disable focusability on the ComposeView wrapper:

```kotlin
class ComposeViewHolder(private val composeView: ComposeView) :
    RecyclerView.ViewHolder(composeView) {

    init {
        // Prevent ComposeView from stealing focus
        composeView.isFocusable = false
        composeView.isFocusableInTouchMode = false
    }

    fun bind(item: Item) {
        composeView.setContent {
            ItemContent(
                item = item,
                modifier = Modifier.semantics(mergeDescendants = true) {
                    // Compose content handles its own semantics
                }
            )
        }
    }
}
```

## Nested Scrollable Containers

### The Problem

Nested scrollable containers confuse TalkBack's scroll detection:

```kotlin
// Problem: Which scroll action applies?
LazyColumn {
    item {
        LazyRow {  // Nested horizontal scroll
            items(10) { HorizontalCard(it) }
        }
    }
    items(20) { VerticalItem(it) }
}
```

### The Fix

Use traversal groups to separate scroll regions:

```kotlin
LazyColumn(
    modifier = Modifier.semantics { isTraversalGroup = true }
) {
    item {
        LazyRow(
            modifier = Modifier.semantics {
                isTraversalGroup = true
                contentDescription = "Featured items, swipe left or right"
            }
        ) {
            items(10) { HorizontalCard(it) }
        }
    }
    items(20) { VerticalItem(it) }
}
```

## Quick Checklist

### RecyclerView

- [ ] `importantForAccessibility="yes"` on RecyclerView
- [ ] `descendantFocusability="afterDescendants"` for carousels
- [ ] Stable item IDs for ViewHolders (`setHasStableIds(true)`)
- [ ] AccessibilityDelegate on ViewHolder if custom behavior needed

### Compose Lazy Lists

- [ ] Stable keys for items: `items(data, key = { it.id })`
- [ ] CollectionInfo on container
- [ ] CollectionItemInfo on items
- [ ] isTraversalGroup for nested scrollables

### Pagers

- [ ] FocusRequester for focus restoration
- [ ] Page position announcements
- [ ] Tab accessibility if using TabLayout

See `references/recyclerview_checklist.md` for detailed ViewHolder patterns.
See `references/pager_patterns.md` for HorizontalPager/ViewPager2 patterns.
