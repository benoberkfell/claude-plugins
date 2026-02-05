# Dual Focus Model Deep Dive

Detailed reference for understanding Android's two independent focus systems.

## Input Focus vs Accessibility Focus

### Input Focus

**What it is**: The element that receives keyboard, D-pad, and hardware input events.

**Controlled by**:
- `View.requestFocus()`
- `FocusRequester.requestFocus()` in Compose
- User D-pad/Tab navigation
- `android:focusable="true"` attribute

**Visual indicator**: Green rectangle (when "Show layout bounds" enabled)

**Relevant APIs**:
```kotlin
// Views
view.isFocused
view.requestFocus()
view.clearFocus()
view.setOnFocusChangeListener { v, hasFocus -> }

// Compose
Modifier.focusable()
Modifier.focusRequester(requester)
Modifier.onFocusChanged { focusState -> }
val focusManager = LocalFocusManager.current
focusManager.moveFocus(FocusDirection.Next)
```

### Accessibility Focus

**What it is**: The element TalkBack is currently reading/highlighting.

**Controlled by**:
- User TalkBack swipe gestures
- `performAction(ACTION_ACCESSIBILITY_FOCUS)` (rare)
- TalkBack's internal focus management
- Node becoming visible/invisible

**Visual indicator**: Green rectangle with thick orange border (TalkBack)

**Relevant APIs**:
```kotlin
// Views
accessibilityNodeInfo.isAccessibilityFocused
view.performAccessibilityAction(
    AccessibilityNodeInfo.ACTION_ACCESSIBILITY_FOCUS,
    null
)

// Compose - Limited direct control
// Focus follows semantics node visibility and user navigation
Modifier.semantics {
    // Properties affect WHICH nodes can receive a11y focus
    // but don't directly control focus
}
```

## Focus State Matrix

| Input Focus | A11y Focus | Scenario |
|-------------|------------|----------|
| TextField | TextField | User typing with TalkBack - synced |
| TextField | Button | User focused TextField, TalkBack exploring elsewhere |
| None | Button | Touch mode, TalkBack on Button |
| Button | None | D-pad focused Button, TalkBack off |

## Synchronization Strategies

### Strategy 1: Let System Handle It

Most cases work automatically. TalkBack moves a11y focus when:
- User swipes to navigate
- Focused View gains input focus AND is accessible
- New content appears with `liveRegion`

```kotlin
// Usually "just works"
TextField(
    modifier = Modifier.focusRequester(focusRequester)
)

LaunchedEffect(showError) {
    if (showError) {
        focusRequester.requestFocus()  // Input focus
        // TalkBack will typically follow for TextField
    }
}
```

### Strategy 2: Announce for Critical Updates

When you need guaranteed TalkBack attention:

```kotlin
// Force announcement regardless of focus
val view = LocalView.current
LaunchedEffect(errorMessage) {
    errorMessage?.let {
        view.announceForAccessibility(it)
    }
}
```

### Strategy 3: Live Region for Dynamic Content

```kotlin
// Content changes trigger announcement
Text(
    text = statusMessage,
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite  // Or Assertive
    }
)
```

## Common Desynchronization Bugs

### Bug 1: Dialog Opens, Focus Stuck Outside

```kotlin
// Problem
AlertDialog(
    onDismissRequest = { },
    text = { Text("Message") },
    confirmButton = { Button(onClick = {}) { Text("OK") } }
)
// Input focus may stay on triggering button

// Solution: Material Dialog handles this
// For custom dialogs, ensure proper focus:
Dialog(
    properties = DialogProperties(
        usePlatformDefaultWidth = false
    )
) {
    Box(Modifier.focusable()) {  // Trap focus in dialog
        // Dialog content
    }
}
```

### Bug 2: Keyboard Opens, TalkBack Loses Position

```kotlin
// Problem: Soft keyboard changes layout, confuses TalkBack

// Solution: Ensure TextField stays visible and focused
val scrollState = rememberScrollState()
val focusRequester = remember { FocusRequester() }

Column(
    modifier = Modifier
        .verticalScroll(scrollState)
        .imePadding()  // Critical!
) {
    TextField(
        modifier = Modifier
            .focusRequester(focusRequester)
            .onFocusChanged {
                if (it.isFocused) {
                    // Scroll to keep visible
                }
            }
    )
}
```

### Bug 3: RecyclerView Recycles Focused View

```kotlin
// Problem: ViewHolder recycled while having a11y focus

// In Adapter
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    // Don't: holder.itemView.requestFocus()

    // Accessibility focus will naturally move
    // when items scroll in/out of view
}

// For focus restoration after scroll, see focus-restoration skill
```

## AccessibilityNodeInfo Lifecycle

Understanding when nodes exist helps debug focus issues:

```
View attached to window
        │
        ▼
onInitializeAccessibilityNodeInfo() called
        │
        ▼
Node added to accessibility tree
        │
        ▼
Node can receive accessibility focus
        │
        ▼
View detached/recycled/GONE
        │
        ▼
Node removed from tree
        │
        ▼
Accessibility focus moves to nearest valid node
```

### Compose Node Lifecycle

```
Composition enters tree
        │
        ▼
semantics { } evaluated
        │
        ▼
SemanticsNode created
        │
        ▼
AndroidComposeView exposes to AccessibilityNodeProvider
        │
        ▼
Node appears in accessibility tree
        │
        ▼
Composition leaves tree (conditional, navigation, etc.)
        │
        ▼
Node removed, focus moves
```

## Debugging Checklist

### Input Focus Not Working

1. Is element `focusable`?
   ```kotlin
   Modifier.focusable()  // Compose
   android:focusable="true"  // XML
   ```

2. Is focus being blocked?
   ```kotlin
   // Check parent doesn't have descendantFocusability="blocksDescendants"
   // Check no other element is requesting focus
   ```

3. Is window focused?
   ```kotlin
   val windowInfo = LocalWindowInfo.current
   Log.d("Focus", "Window focused: ${windowInfo.isWindowFocused}")
   ```

### Accessibility Focus Not Working

1. Does element have semantics?
   ```kotlin
   Modifier.semantics { contentDescription = "..." }
   ```

2. Is element hidden from accessibility?
   ```kotlin
   // Check for invisibleToUser(), clearAndSetSemantics on parent
   ```

3. Is TalkBack actually enabled?
   ```kotlin
   val am = context.getSystemService(AccessibilityManager::class.java)
   Log.d("A11y", "Touch exploration: ${am.isTouchExplorationEnabled}")
   ```

4. Print the tree to verify node exists:
   ```kotlin
   composeTestRule.onRoot().printToLog("FOCUS_DEBUG")
   ```

## Platform Differences

| Behavior | Android 11- | Android 12+ |
|----------|-------------|-------------|
| Focus indicator | Green box | Green + orange |
| Focus ordering | Geometric only | traversalIndex honored |
| Initial focus | First focusable | Respects requestFocus |
| Focus in WebView | Limited | Improved bridge |

## Related Skills

- `android-a11y-focus-restoration` - Preserving focus across navigation
- `android-a11y-troubleshooting` - Fixing focus-related bugs
- `android-a11y-hybrid-debugging` - Focus in View/Compose interop
