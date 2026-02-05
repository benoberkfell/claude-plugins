# View-Based Announcements

Reference for making dynamic announcements in View-based Android code.

## Announcement Methods Comparison

| Method | Use Case | Interrupts Speech | Timing Control |
|--------|----------|-------------------|----------------|
| `announceForAccessibility()` | One-time announcements | No | Immediate |
| Live Region (Polite) | Dynamic content updates | No | After current |
| Live Region (Assertive) | Critical alerts | Yes | Immediate |
| AccessibilityDelegate | Custom announcements | Configurable | Full control |
| Custom Toast | Visual + audio feedback | No | System-managed |

## announceForAccessibility()

### Basic Usage

```kotlin
// Simple announcement
view.announceForAccessibility("Item deleted")

// From Activity/Fragment
requireActivity().window.decorView
    .announceForAccessibility("Loading complete")
```

### When to Use

- One-time status updates
- Action confirmations
- Error messages
- Loading state changes

### Example: Form Submission

```kotlin
fun submitForm() {
    // Start loading
    submitButton.announceForAccessibility("Submitting form")

    viewModel.submit().observe(this) { result ->
        when (result) {
            is Success -> {
                rootView.announceForAccessibility("Form submitted successfully")
                navigateToConfirmation()
            }
            is Error -> {
                rootView.announceForAccessibility("Error: ${result.message}")
                errorText.requestFocus()
            }
        }
    }
}
```

## AccessibilityDelegate for Custom Announcements

### Basic Setup

```kotlin
class AnnouncingView(context: Context) : View(context) {

    init {
        accessibilityDelegate = object : AccessibilityDelegate() {

            override fun onInitializeAccessibilityNodeInfo(
                host: View,
                info: AccessibilityNodeInfo
            ) {
                super.onInitializeAccessibilityNodeInfo(host, info)
                // Customize node info
                info.contentDescription = getCurrentDescription()
            }

            override fun sendAccessibilityEvent(host: View, eventType: Int) {
                // Intercept and customize events
                if (eventType == AccessibilityEvent.TYPE_VIEW_CLICKED) {
                    // Custom announcement on click
                    host.announceForAccessibility("Custom action performed")
                }
                super.sendAccessibilityEvent(host, eventType)
            }
        }
    }
}
```

### Custom Event Handling

```kotlin
class CustomButtonDelegate(
    private val onClickAnnouncement: () -> String
) : View.AccessibilityDelegate() {

    override fun sendAccessibilityEvent(host: View, eventType: Int) {
        when (eventType) {
            AccessibilityEvent.TYPE_VIEW_CLICKED -> {
                // Replace default announcement
                val event = AccessibilityEvent.obtain(eventType).apply {
                    text.add(onClickAnnouncement())
                }
                host.parent?.requestSendAccessibilityEvent(host, event)
                return  // Don't call super - we handled it
            }
        }
        super.sendAccessibilityEvent(host, eventType)
    }
}

// Usage
button.accessibilityDelegate = CustomButtonDelegate {
    if (isLoading) "Please wait, loading" else "Submit form"
}
```

## Live Regions in Views

### XML Declaration

```xml
<TextView
    android:id="@+id/statusText"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:accessibilityLiveRegion="polite" />

<!-- For critical updates -->
<TextView
    android:id="@+id/errorText"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:accessibilityLiveRegion="assertive" />
```

### Programmatic Setup

```kotlin
statusText.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_POLITE
errorText.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_ASSERTIVE
```

### Live Region Modes

**POLITE** (`View.ACCESSIBILITY_LIVE_REGION_POLITE`):
- Waits for current speech to finish
- Good for status updates, progress
- Non-disruptive

**ASSERTIVE** (`View.ACCESSIBILITY_LIVE_REGION_ASSERTIVE`):
- Interrupts current speech immediately
- Good for errors, critical alerts
- Use sparingly to avoid annoyance

### Example: Progress Updates

```kotlin
class ProgressManager(private val progressText: TextView) {

    init {
        progressText.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_POLITE
    }

    fun updateProgress(percent: Int) {
        // Only announce at meaningful intervals
        val shouldAnnounce = percent % 25 == 0 || percent == 100

        progressText.text = "$percent% complete"

        if (!shouldAnnounce) {
            // Temporarily disable live region for minor updates
            progressText.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_NONE
            progressText.post {
                progressText.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_POLITE
            }
        }
    }
}
```

## Custom Toast Accessibility

### The Problem

Custom Toast views don't get TalkBack announcements like system Toasts.

```kotlin
// Problem: Custom toast is silent to TalkBack
val customToast = Toast(context).apply {
    view = LayoutInflater.from(context).inflate(R.layout.custom_toast, null)
    duration = Toast.LENGTH_SHORT
}
customToast.show()
```

### Solution 1: Manual Announcement

```kotlin
fun showAccessibleToast(message: String) {
    // Show visual toast
    val customToast = Toast(context).apply {
        view = createCustomToastView(message)
        duration = Toast.LENGTH_SHORT
    }
    customToast.show()

    // Announce for accessibility
    val rootView = (context as Activity).window.decorView
    rootView.announceForAccessibility(message)
}
```

### Solution 2: Snackbar (Recommended)

Snackbar has built-in accessibility support:

```kotlin
Snackbar.make(rootView, "Item deleted", Snackbar.LENGTH_SHORT)
    .setAction("Undo") { undoDelete() }
    .show()
// TalkBack automatically announces this
```

### Solution 3: AccessibilityManager Direct

```kotlin
fun announceMessage(context: Context, message: String) {
    val accessibilityManager = context.getSystemService(Context.ACCESSIBILITY_SERVICE)
        as AccessibilityManager

    if (accessibilityManager.isEnabled) {
        val event = AccessibilityEvent.obtain().apply {
            eventType = AccessibilityEvent.TYPE_ANNOUNCEMENT
            text.add(message)
        }
        accessibilityManager.sendAccessibilityEvent(event)
    }
}
```

## Timing Considerations

### Delayed Announcements

```kotlin
fun announceAfterAnimation(view: View, message: String, delayMs: Long = 300) {
    view.postDelayed({
        view.announceForAccessibility(message)
    }, delayMs)
}
```

### Debounced Announcements

```kotlin
class DebouncedAnnouncer(
    private val view: View,
    private val debounceMs: Long = 500
) {
    private var lastAnnouncement: String? = null
    private var pendingRunnable: Runnable? = null
    private val handler = Handler(Looper.getMainLooper())

    fun announce(message: String) {
        if (message == lastAnnouncement) return

        pendingRunnable?.let { handler.removeCallbacks(it) }

        pendingRunnable = Runnable {
            view.announceForAccessibility(message)
            lastAnnouncement = message
        }.also {
            handler.postDelayed(it, debounceMs)
        }
    }

    fun announceImmediately(message: String) {
        pendingRunnable?.let { handler.removeCallbacks(it) }
        view.announceForAccessibility(message)
        lastAnnouncement = message
    }
}
```

## RecyclerView Announcements

### Item Change Announcements

```kotlin
adapter.registerAdapterDataObserver(object : RecyclerView.AdapterDataObserver() {

    override fun onItemRangeInserted(positionStart: Int, itemCount: Int) {
        val message = when (itemCount) {
            1 -> "Item added"
            else -> "$itemCount items added"
        }
        recyclerView.announceForAccessibility(message)
    }

    override fun onItemRangeRemoved(positionStart: Int, itemCount: Int) {
        val message = when (itemCount) {
            1 -> "Item removed"
            else -> "$itemCount items removed"
        }
        recyclerView.announceForAccessibility(message)
    }

    override fun onItemRangeChanged(positionStart: Int, itemCount: Int) {
        // Usually don't announce range changes to avoid noise
    }
})
```

### Empty State Announcement

```kotlin
fun updateEmptyState(isEmpty: Boolean) {
    if (isEmpty) {
        emptyView.visibility = View.VISIBLE
        recyclerView.visibility = View.GONE

        // Announce empty state
        emptyView.announceForAccessibility("List is empty")
    } else {
        emptyView.visibility = View.GONE
        recyclerView.visibility = View.VISIBLE
    }
}
```

## Best Practices

### Do

- Announce meaningful state changes
- Use Polite for most updates, Assertive for errors
- Keep announcements concise
- Test with TalkBack enabled

### Don't

- Announce every minor change (causes noise)
- Use Assertive for non-critical updates
- Announce what's already visible/obvious
- Forget to test timing with animations

### Announcement Content Guidelines

```kotlin
// Good: Concise, actionable
view.announceForAccessibility("3 items selected")
view.announceForAccessibility("Error: Invalid email address")
view.announceForAccessibility("Loading complete, 10 results")

// Bad: Too verbose, redundant
view.announceForAccessibility("The system has detected that you have selected 3 items from the list")
view.announceForAccessibility("Error error error! The email is wrong!")
```
