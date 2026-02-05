---
name: android-a11y-focus-architecture
description: Understand how Android accessibility focus works at the system level. Use when debugging why focus isn't working, understanding the dual-focus model, or diagnosing focus desynchronization issues.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Android Accessibility Focus Architecture

This skill explains how accessibility focus works under the hood - essential knowledge for debugging complex focus issues.

## The Dual-Focus Model

Android has **two independent focus systems**:

1. **Input Focus** - Where keyboard/D-pad input goes (green rectangle in developer options)
2. **Accessibility Focus** - Where TalkBack is currently reading (green rectangle with orange border)

These can point to different elements simultaneously.

```
┌─────────────────────────────────────────────────────────┐
│                    Android System                        │
│  ┌─────────────────┐         ┌─────────────────────┐   │
│  │   Input Focus   │         │ Accessibility Focus │   │
│  │   (keyboard)    │         │    (TalkBack)       │   │
│  └────────┬────────┘         └──────────┬──────────┘   │
│           │                             │              │
│           ▼                             ▼              │
│    View.isFocused()            AccessibilityNodeInfo   │
│                                .isAccessibilityFocused │
└─────────────────────────────────────────────────────────┘
```

### Why They Desynchronize

```kotlin
// Common cause: Input focus moves but a11y focus doesn't
editText.requestFocus()  // Input focus moves
// TalkBack still on previous element!

// Solution: Request both
editText.requestFocus()
editText.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)
```

### In Compose

```kotlin
// FocusRequester controls INPUT focus, not accessibility focus
val focusRequester = remember { FocusRequester() }

TextField(
    modifier = Modifier.focusRequester(focusRequester)
)

LaunchedEffect(Unit) {
    focusRequester.requestFocus()  // Input focus only
}
```

For accessibility focus in Compose, you typically rely on TalkBack's natural navigation rather than programmatic control.

## The Accessibility Node Tree

TalkBack doesn't see your View hierarchy directly. It sees a **simplified accessibility tree** built from `AccessibilityNodeInfo` objects.

```
View Hierarchy                    Accessibility Tree (what TalkBack sees)
─────────────────                 ──────────────────────────────────────
ConstraintLayout                  [merged into parent]
├── ImageView                     Node: "Profile photo", Role.Image
├── LinearLayout                  [merged - mergeDescendants]
│   ├── TextView "Name"      ──►  └── Node: "Name, Software Engineer"
│   └── TextView "Title"              (merged text)
└── Button "Edit"                 Node: "Edit", Role.Button, onClick

```

### Key Differences

| View Hierarchy | Accessibility Tree |
|---------------|-------------------|
| All views | Only semantic nodes |
| Nested deeply | Flattened structure |
| Layout containers | Merged/invisible |
| Every TextView | Merged text nodes |

### How Nodes Get Created

```kotlin
// In Views: Automatic for standard widgets
// Override for custom views:
class CustomView : View {
    override fun onInitializeAccessibilityNodeInfo(info: AccessibilityNodeInfo) {
        super.onInitializeAccessibilityNodeInfo(info)
        info.className = Button::class.java.name
        info.contentDescription = "Custom action"
        info.addAction(AccessibilityNodeInfo.AccessibilityAction.ACTION_CLICK)
    }
}

// In Compose: semantics modifier
Modifier.semantics {
    contentDescription = "Custom action"
    role = Role.Button
    onClick { performAction(); true }
}
```

## Focus Search Algorithm

When you swipe in TalkBack, it runs a **focus search** to find the next node:

1. Get all focusable nodes in the accessibility tree
2. Sort by traversal order (geometric position or `traversalIndex`)
3. Find current node in sorted list
4. Return next/previous based on swipe direction

### Default Traversal Order

```
┌──────────────────────────────┐
│  1 ─────► 2 ─────► 3        │  Left-to-right, top-to-bottom
│                              │  (in LTR locales)
│  4 ─────► 5 ─────► 6        │
│                              │
│  7 ─────► 8 ─────► 9        │
└──────────────────────────────┘
```

### When Default Fails

Complex layouts can confuse the geometric algorithm:

```kotlin
// Problem: Overlapping elements
Box {
    Card(Modifier.fillMaxSize()) { ... }  // Background
    FloatingActionButton(                  // Overlapping
        Modifier.align(Alignment.BottomEnd)
    ) { ... }
}
// TalkBack may read FAB at wrong time

// Solution: Explicit traversal order
Box(Modifier.semantics { isTraversalGroup = true }) {
    Card(Modifier.semantics { traversalIndex = 0f }) { ... }
    FloatingActionButton(
        Modifier.semantics { traversalIndex = 1f }
    ) { ... }
}
```

## Initial Focus on Screen Entry

When a new screen appears, TalkBack uses heuristics to find the first element:

1. Look for element with explicit `requestFocus`
2. Find topmost-leftmost focusable element
3. Fall back to first element in tree

### XML: requestFocus

```xml
<EditText
    android:id="@+id/username"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">
    <requestFocus />
</EditText>
```

### Compose: FocusRequester with LaunchedEffect

```kotlin
@Composable
fun LoginScreen() {
    val focusRequester = remember { FocusRequester() }

    LaunchedEffect(Unit) {
        // Delay to ensure composition is complete
        delay(100)
        focusRequester.requestFocus()
    }

    Column {
        Text(
            "Welcome",
            modifier = Modifier.semantics { heading() }
        )
        TextField(
            value = username,
            onValueChange = { username = it },
            modifier = Modifier.focusRequester(focusRequester)
        )
    }
}
```

### Force Initial Accessibility Focus

For critical announcements on screen entry:

```kotlin
@Composable
fun AlertScreen(message: String) {
    val view = LocalView.current

    LaunchedEffect(Unit) {
        view.announceForAccessibility(message)
    }

    Text(
        text = message,
        modifier = Modifier.semantics {
            liveRegion = LiveRegionMode.Assertive
        }
    )
}
```

## Common Focus Architecture Issues

### Issue 1: Focus Lost After Recomposition

```kotlin
// Problem: Key changes cause new node
key(itemId) {  // Different key = different node
    FocusableItem(item)
}
// A11y focus resets to start

// Solution: Stable keys or no key
items(items, key = { it.stableId }) { item ->
    FocusableItem(item)
}
```

### Issue 2: Focus Trapped in Container

```kotlin
// Problem: Container blocks focus escape
ScrollView(
    modifier = Modifier.focusable()  // Traps focus!
) {
    Column { ... }
}

// Solution: Don't make containers focusable
ScrollView {  // Natural scrolling works
    Column { ... }
}
```

### Issue 3: AndroidView Focus Interop

```kotlin
// Problem: Compose and View focus fight
AndroidView(
    factory = { context ->
        EditText(context).apply {
            // View wants its own focus management
        }
    },
    modifier = Modifier.focusable()  // Conflict!
)

// Solution: Let View handle its focus
AndroidView(
    factory = { context ->
        EditText(context)
    }
    // Don't add focusable modifier
)
```

## Debugging Focus State

### Check Input Focus

```kotlin
// In app code
val windowInfo = LocalWindowInfo.current
val hasFocus = windowInfo.isWindowFocused

// Via ADB
adb shell dumpsys input | grep -A 20 "FocusController"
```

### Check Accessibility Focus

```kotlin
// Print which node has a11y focus
val accessibilityManager = context.getSystemService(AccessibilityManager::class.java)
Log.d("A11y", "Enabled: ${accessibilityManager.isEnabled}")
Log.d("A11y", "TalkBack: ${accessibilityManager.isTouchExplorationEnabled}")

// Via ADB
adb shell dumpsys accessibility | grep -A 10 "currentFocus"
```

### Visualize Focus

Enable in Developer Options:
- **Show layout bounds** - See view boundaries
- **Pointer location** - See touch coordinates
- **Show taps** - Verify touch registration

See `references/dual_focus_model.md` for detailed diagrams and edge cases.
