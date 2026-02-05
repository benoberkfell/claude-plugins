---
name: android-a11y-hybrid-debugging
description: Debug accessibility in hybrid View/Compose apps and use system-level diagnostic tools. Use when focus jumps between Compose and Views unexpectedly, WebViews are inaccessible, or you need ADB-level debugging.
allowed-tools: Read, Glob, Grep, Edit, Write, Bash
---

# Hybrid Architecture Accessibility Debugging

This skill covers accessibility issues in hybrid View/Compose apps and system-level debugging techniques.

## ComposeView in RecyclerView

### The Problem

When mixing `ComposeView` inside RecyclerView items, the `AndroidComposeView` can steal accessibility focus from sibling View-based items.

```kotlin
// Problem: Focus jumps unpredictably between View and Compose items
class MixedAdapter : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) = when (viewType) {
        TYPE_COMPOSE -> ComposeViewHolder(ComposeView(parent.context))
        TYPE_VIEW -> ViewViewHolder(LayoutInflater.inflate(R.layout.item_view, parent, false))
        else -> throw IllegalArgumentException()
    }
}
```

### The Fix

Disable focusability on the ComposeView wrapper:

```kotlin
class ComposeViewHolder(private val composeView: ComposeView) :
    RecyclerView.ViewHolder(composeView) {

    init {
        // Critical: Prevent ComposeView container from stealing focus
        composeView.apply {
            isFocusable = false
            isFocusableInTouchMode = false
            // Let Compose content handle its own accessibility
            importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS
        }
    }

    fun bind(item: ComposeItem) {
        composeView.setContent {
            // Compose content exposes its own semantics
            ItemContent(
                item = item,
                modifier = Modifier.semantics(mergeDescendants = true) {
                    contentDescription = item.description
                }
            )
        }
    }
}
```

### Alternative: Compose-First Approach

If possible, use Compose for the entire list:

```kotlin
@Composable
fun MixedContentList(items: List<Any>) {
    LazyColumn {
        items(items, key = { it.hashCode() }) { item ->
            when (item) {
                is ComposeItem -> ComposeItemRow(item)
                is LegacyItem -> AndroidViewItem(item)  // Wrap legacy View
            }
        }
    }
}

@Composable
fun AndroidViewItem(item: LegacyItem) {
    AndroidView(
        factory = { context ->
            LayoutInflater.from(context)
                .inflate(R.layout.legacy_item, null)
        },
        update = { view ->
            view.findViewById<TextView>(R.id.title).text = item.title
            // Legacy view handles its own accessibility
        },
        modifier = Modifier.semantics {
            // Override if legacy view accessibility is insufficient
            contentDescription = item.accessibilityDescription
        }
    )
}
```

## WebView Accessibility

### The Problem

WebView creates a separate accessibility tree from its DOM, which can appear as a "black hole" to TalkBack if not configured properly.

### Basic WebView Setup

```kotlin
class AccessibleWebViewActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        webView.apply {
            // Enable accessibility
            settings.javaScriptEnabled = true

            // Critical for accessibility
            importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

            // Enable DOM storage for modern web pages
            settings.domStorageEnabled = true

            // Set a content description for the WebView container
            contentDescription = "Web content"
        }
    }
}
```

### WebView in RecyclerView

```kotlin
class WebViewViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    private val webView: WebView = itemView.findViewById(R.id.webView)

    init {
        webView.apply {
            importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

            // Prevent WebView from blocking RecyclerView scrolling
            isNestedScrollingEnabled = false

            // Set accessibility delegate for custom behavior
            accessibilityDelegate = object : View.AccessibilityDelegate() {
                override fun onInitializeAccessibilityNodeInfo(
                    host: View,
                    info: AccessibilityNodeInfo
                ) {
                    super.onInitializeAccessibilityNodeInfo(host, info)
                    info.className = WebView::class.java.name
                    // Add hint about web content
                    info.hintText = "Double tap to explore web content"
                }
            }
        }
    }

    fun bind(url: String) {
        webView.loadUrl(url)
    }
}
```

### Compose WebView Integration

```kotlin
@Composable
fun AccessibleWebView(
    url: String,
    modifier: Modifier = Modifier
) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true
                importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        },
        modifier = modifier.semantics {
            contentDescription = "Web page content"
            // Mark as traversal group to contain web accessibility tree
            isTraversalGroup = true
        }
    )
}
```

## ADB Debugging Commands

### Accessibility Service State

```bash
# Check accessibility services enabled
adb shell settings get secure enabled_accessibility_services

# Check if TalkBack is running
adb shell dumpsys accessibility | grep -A 5 "Service\[label=TalkBack"

# Full accessibility dump
adb shell dumpsys accessibility
```

### Focus State Analysis

```bash
# Check current accessibility focus
adb shell dumpsys accessibility | grep -A 10 "mAccessibilityFocusedWindow"

# Check input focus
adb shell dumpsys input | grep -A 20 "FocusedWindow"

# Window focus state
adb shell dumpsys window | grep -E "mCurrentFocus|mFocusedApp"
```

### Node Tree Debugging

```bash
# Enable verbose accessibility logging
adb shell setprop log.tag.SemanticsOwner VERBOSE
adb shell setprop log.tag.AccessibilityNodeInfo VERBOSE

# Filter TalkBack logcat
adb logcat | grep -E "TalkBack|SemanticsOwner|AccessibilityNode"

# Compose semantics debugging
adb logcat | grep -E "SemanticsOwner|ComposeAccessibility"
```

### Node Info Dump

```bash
# Dump accessibility node info for focused window
adb shell uiautomator dump /sdcard/ui.xml && adb pull /sdcard/ui.xml

# Interactive exploration
adb shell uiautomator events  # Watch accessibility events live
```

## Layout Inspector Debugging

### Using Layout Inspector

1. Open Android Studio > View > Tool Windows > Layout Inspector
2. Select your running process
3. Click "Show accessibility info" in the toolbar

### What to Look For

- **Semantics Tree** - Compare merged vs unmerged
- **Content Description** - Verify correct text
- **Actions** - Ensure onClick, etc. are present
- **Traversal Order** - Check `traversalIndex` values

### Compose Semantics in Layout Inspector

```
Node hierarchy shows:
├── ComposeView
│   └── AndroidComposeView
│       └── SemanticsNode (root)
│           ├── Text "Title" [heading]
│           ├── Button "Submit" [onClick]
│           └── ...
```

## TreeDebug Logging

### Enable in Compose

```kotlin
@Composable
fun DebugScreen() {
    val view = LocalView.current

    LaunchedEffect(Unit) {
        // Print semantics tree to logcat
        if (view is AndroidComposeView) {
            Log.d("SemanticsDebug",
                view.semanticsOwner.rootSemanticsNode.printToString())
        }
    }

    // Or in tests
    composeTestRule.onRoot().printToLog("TREE_DEBUG")
}
```

### Logcat Filters

```bash
# Semantics tree changes
adb logcat -s SemanticsOwner:V

# Compose focus events
adb logcat | grep -E "FocusTraversal|FocusState"

# Accessibility events
adb logcat | grep -E "AccessibilityEvent|TYPE_VIEW"
```

## Systematic Debugging Process

### Step 1: Identify the Layer

```
Issue in...
├── Compose semantics? → Print tree, check semantics modifiers
├── View accessibility? → Check AccessibilityNodeInfo, delegate
├── Bridge (interop)? → Check AndroidView/ComposeView setup
└── System service? → Use dumpsys, check TalkBack logs
```

### Step 2: Capture State

```kotlin
// Custom debugging delegate
class DebugAccessibilityDelegate(private val tag: String) : View.AccessibilityDelegate() {

    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfo) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        Log.d(tag, """
            ═══════════════════════════════════
            View: ${host::class.simpleName}
            Class: ${info.className}
            ContentDesc: ${info.contentDescription}
            Text: ${info.text}
            Focusable: ${info.isFocusable}
            A11y Focused: ${info.isAccessibilityFocused}
            Actions: ${info.actionList.map { "${it.id}:${it.label}" }}
            ═══════════════════════════════════
        """.trimIndent())
    }

    override fun sendAccessibilityEvent(host: View, eventType: Int) {
        Log.d(tag, "Event: ${accessibilityEventTypeToString(eventType)} from ${host::class.simpleName}")
        super.sendAccessibilityEvent(host, eventType)
    }

    private fun accessibilityEventTypeToString(type: Int) = when (type) {
        AccessibilityEvent.TYPE_VIEW_CLICKED -> "CLICKED"
        AccessibilityEvent.TYPE_VIEW_FOCUSED -> "FOCUSED"
        AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED -> "A11Y_FOCUSED"
        AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUS_CLEARED -> "A11Y_FOCUS_CLEARED"
        AccessibilityEvent.TYPE_ANNOUNCEMENT -> "ANNOUNCEMENT"
        else -> "TYPE_$type"
    }
}

// Apply to views for debugging
view.accessibilityDelegate = DebugAccessibilityDelegate("MyView")
```

### Step 3: Compare Expected vs Actual

| Check | Expected | Actual | Fix |
|-------|----------|--------|-----|
| Focusable | true | false | Add `focusable` modifier |
| Actions | [onClick] | [] | Add `onClick` in semantics |
| Description | "Submit button" | null | Add `contentDescription` |
| Role | Button | none | Add `role = Role.Button` |

### Step 4: Isolate

```kotlin
// Isolate component in test
@Test
fun isolatedComponentA11y() {
    composeTestRule.setContent {
        // Just the component, no navigation/state
        ProblemComponent(
            data = testData,
            onClick = {}
        )
    }

    // Verify accessibility properties
    composeTestRule.onRoot().printToLog("ISOLATED")

    composeTestRule
        .onNode(hasContentDescription("Expected"))
        .assertExists()
        .assert(hasRole(Role.Button))
}
```

## Common Hybrid Issues Checklist

### ComposeView in RecyclerView
- [ ] ComposeView `isFocusable = false`
- [ ] ComposeView `isFocusableInTouchMode = false`
- [ ] Compose content has proper semantics
- [ ] Stable keys for RecyclerView items

### AndroidView in Compose
- [ ] No conflicting `focusable` modifiers
- [ ] View handles its own accessibility
- [ ] Semantics modifier only if overriding View

### WebView Integration
- [ ] `importantForAccessibility = YES`
- [ ] JavaScript enabled
- [ ] Web page has proper ARIA attributes
- [ ] Container has `isTraversalGroup`

### Focus Issues
- [ ] Check dual-focus model (input vs a11y focus)
- [ ] Verify focus requests happen after composition
- [ ] No competing focus requests

See `references/adb_commands.md` for complete ADB reference.
See `references/interop_fixes.md` for hybrid architecture solutions.
