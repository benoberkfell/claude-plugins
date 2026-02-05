# RecyclerView Accessibility Checklist

Complete reference for making RecyclerView accessible.

## RecyclerView Setup

### XML Attributes

```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:importantForAccessibility="yes"
    android:contentDescription="@string/list_description" />
```

### Kotlin Setup

```kotlin
recyclerView.apply {
    // Enable accessibility
    importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

    // For carousels: allow child focus
    descendantFocusability = ViewGroup.FOCUS_AFTER_DESCENDANTS

    // Enable stable IDs for better recycling behavior
    adapter?.setHasStableIds(true)
}
```

## Adapter Patterns

### Basic Accessible Adapter

```kotlin
class AccessibleAdapter(
    private val items: List<Item>,
    private val onItemClick: (Item) -> Unit
) : RecyclerView.Adapter<AccessibleAdapter.ViewHolder>() {

    init {
        setHasStableIds(true)
    }

    override fun getItemId(position: Int) = items[position].id

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_row, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(items[position], position, itemCount)
    }

    override fun getItemCount() = items.size

    inner class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        private val titleText: TextView = itemView.findViewById(R.id.title)
        private val subtitleText: TextView = itemView.findViewById(R.id.subtitle)

        fun bind(item: Item, position: Int, total: Int) {
            titleText.text = item.title
            subtitleText.text = item.subtitle

            // Set accessibility info
            itemView.contentDescription = buildString {
                append(item.title)
                append(", ")
                append(item.subtitle)
                append(". ")
                append("Item ${position + 1} of $total")
            }

            itemView.setOnClickListener {
                onItemClick(item)
            }
        }
    }
}
```

### Custom AccessibilityDelegate on ViewHolder

```kotlin
class EnhancedViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {

    init {
        itemView.accessibilityDelegate = object : View.AccessibilityDelegate() {
            override fun onInitializeAccessibilityNodeInfo(
                host: View,
                info: AccessibilityNodeInfo
            ) {
                super.onInitializeAccessibilityNodeInfo(host, info)

                // Add collection item info
                info.setCollectionItemInfo(
                    AccessibilityNodeInfo.CollectionItemInfo.obtain(
                        bindingAdapterPosition,  // rowIndex
                        1,                       // rowSpan
                        0,                       // columnIndex
                        1,                       // columnSpan
                        false                    // isHeading
                    )
                )

                // Add custom actions
                info.addAction(
                    AccessibilityNodeInfo.AccessibilityAction(
                        AccessibilityNodeInfo.ACTION_CLICK,
                        "Open details"
                    )
                )

                info.addAction(
                    AccessibilityNodeInfo.AccessibilityAction(
                        R.id.action_delete,
                        "Delete item"
                    )
                )
            }

            override fun performAccessibilityAction(
                host: View,
                action: Int,
                args: Bundle?
            ): Boolean {
                return when (action) {
                    R.id.action_delete -> {
                        // Handle delete
                        true
                    }
                    else -> super.performAccessibilityAction(host, action, args)
                }
            }
        }
    }
}
```

## Collection Info on RecyclerView

```kotlin
recyclerView.accessibilityDelegate = object : View.AccessibilityDelegate() {
    override fun onInitializeAccessibilityNodeInfo(
        host: View,
        info: AccessibilityNodeInfo
    ) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        val adapter = (host as RecyclerView).adapter ?: return
        val layoutManager = host.layoutManager

        val rowCount: Int
        val columnCount: Int

        when (layoutManager) {
            is LinearLayoutManager -> {
                if (layoutManager.orientation == LinearLayoutManager.VERTICAL) {
                    rowCount = adapter.itemCount
                    columnCount = 1
                } else {
                    rowCount = 1
                    columnCount = adapter.itemCount
                }
            }
            is GridLayoutManager -> {
                rowCount = (adapter.itemCount + layoutManager.spanCount - 1) /
                    layoutManager.spanCount
                columnCount = layoutManager.spanCount
            }
            else -> {
                rowCount = adapter.itemCount
                columnCount = 1
            }
        }

        info.setCollectionInfo(
            AccessibilityNodeInfo.CollectionInfo.obtain(
                rowCount,
                columnCount,
                false  // isHierarchical
            )
        )
    }
}
```

## Handling Item Updates

### Announce Changes

```kotlin
adapter.registerAdapterDataObserver(object : RecyclerView.AdapterDataObserver() {
    override fun onItemRangeInserted(positionStart: Int, itemCount: Int) {
        recyclerView.announceForAccessibility(
            "$itemCount items added"
        )
    }

    override fun onItemRangeRemoved(positionStart: Int, itemCount: Int) {
        recyclerView.announceForAccessibility(
            "$itemCount items removed"
        )
    }
})
```

### Preserve Focus After Update

```kotlin
// Before update: save focused position
val focusedPosition = recyclerView.findViewHolderForAdapterPosition(
    layoutManager.findFirstVisibleItemPosition()
)?.bindingAdapterPosition

// After update: restore focus
focusedPosition?.let { position ->
    recyclerView.post {
        recyclerView.findViewHolderForAdapterPosition(position)
            ?.itemView
            ?.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
    }
}
```

## Horizontal Carousel Pattern

```kotlin
class CarouselAdapter(
    private val items: List<CarouselItem>
) : RecyclerView.Adapter<CarouselAdapter.ViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        return ViewHolder(
            LayoutInflater.from(parent.context)
                .inflate(R.layout.carousel_item, parent, false)
        )
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val item = items[position]

        holder.itemView.apply {
            // Make item focusable
            isFocusable = true

            // Set content description with position context
            contentDescription = buildString {
                append(item.title)
                append(". ")
                append("${position + 1} of ${items.size}")
                if (position == 0) append(". Swipe right for more.")
            }
        }
    }

    override fun getItemCount() = items.size

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView)
}

// RecyclerView setup for carousel
horizontalRecyclerView.apply {
    layoutManager = LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false)
    adapter = CarouselAdapter(items)

    // Critical for carousel accessibility
    descendantFocusability = ViewGroup.FOCUS_AFTER_DESCENDANTS
    importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

    // Announce as carousel
    contentDescription = "Featured items carousel"
}
```

## Expandable Items

```kotlin
class ExpandableViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    private var isExpanded = false

    fun bind(item: ExpandableItem) {
        itemView.apply {
            // Update expanded state
            accessibilityDelegate = object : View.AccessibilityDelegate() {
                override fun onInitializeAccessibilityNodeInfo(
                    host: View,
                    info: AccessibilityNodeInfo
                ) {
                    super.onInitializeAccessibilityNodeInfo(host, info)

                    // Announce expand/collapse state
                    info.addAction(
                        AccessibilityNodeInfo.AccessibilityAction(
                            if (isExpanded)
                                AccessibilityNodeInfo.ACTION_COLLAPSE
                            else
                                AccessibilityNodeInfo.ACTION_EXPAND,
                            if (isExpanded) "Collapse" else "Expand"
                        )
                    )
                }

                override fun performAccessibilityAction(
                    host: View,
                    action: Int,
                    args: Bundle?
                ): Boolean {
                    return when (action) {
                        AccessibilityNodeInfo.ACTION_EXPAND -> {
                            expand()
                            true
                        }
                        AccessibilityNodeInfo.ACTION_COLLAPSE -> {
                            collapse()
                            true
                        }
                        else -> super.performAccessibilityAction(host, action, args)
                    }
                }
            }
        }
    }

    private fun expand() {
        isExpanded = true
        itemView.announceForAccessibility("Expanded")
        // Update UI
    }

    private fun collapse() {
        isExpanded = false
        itemView.announceForAccessibility("Collapsed")
        // Update UI
    }
}
```

## Debugging RecyclerView Accessibility

### Verify Scroll Actions

```kotlin
recyclerView.accessibilityDelegate = object : View.AccessibilityDelegate() {
    override fun onInitializeAccessibilityNodeInfo(
        host: View,
        info: AccessibilityNodeInfo
    ) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        Log.d("A11y", "RecyclerView actions:")
        info.actionList.forEach { action ->
            Log.d("A11y", "  - ${action.id}: ${action.label}")
        }

        // Should include:
        // - 4096: Scroll forward
        // - 8192: Scroll backward
    }
}
```

### Verify ViewHolder Accessibility

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    holder.bind(items[position])

    // Debug: Log accessibility info
    holder.itemView.post {
        val nodeInfo = AccessibilityNodeInfo.obtain()
        holder.itemView.onInitializeAccessibilityNodeInfo(nodeInfo)
        Log.d("A11y", "Position $position:")
        Log.d("A11y", "  contentDescription: ${nodeInfo.contentDescription}")
        Log.d("A11y", "  className: ${nodeInfo.className}")
        Log.d("A11y", "  focusable: ${nodeInfo.isFocusable}")
        Log.d("A11y", "  actions: ${nodeInfo.actionList.map { it.id }}")
        nodeInfo.recycle()
    }
}
```

## Common Issues Quick Reference

| Issue | Check | Fix |
|-------|-------|-----|
| Can't scroll with TalkBack | `importantForAccessibility` | Set to `yes` |
| Focus trapped in carousel | `descendantFocusability` | Set to `afterDescendants` |
| Items not focusable | ViewHolder `isFocusable` | Set to `true` |
| No "in list" announcement | Collection info missing | Add CollectionInfo |
| Item position not announced | CollectionItemInfo missing | Add to each ViewHolder |
| Focus jumps after update | Unstable IDs | Enable `setHasStableIds(true)` |
