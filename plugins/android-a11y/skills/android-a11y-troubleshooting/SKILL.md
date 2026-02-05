---
name: android-a11y-troubleshooting
description: Debug and fix Compose accessibility issues. Use when TalkBack isn't reading content correctly, elements are unreachable, or accessibility tests fail.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Compose Accessibility Troubleshooting

Common accessibility issues and how to fix them.

## TalkBack Not Reading Element

### Symptom
An element is visible but TalkBack skips over it.

### Causes & Solutions

**1. Element has no semantics**
```kotlin
// Problem: Canvas/custom drawing has no semantics
Canvas(modifier = Modifier.fillMaxSize()) {
    drawCircle(Color.Red)
}

// Solution: Add semantics
Canvas(
    modifier = Modifier
        .fillMaxSize()
        .semantics { contentDescription = "Red circle indicator" }
) {
    drawCircle(Color.Red)
}
```

**2. Parent uses clearAndSetSemantics**
```kotlin
// Problem: Parent cleared child semantics
Box(Modifier.clearAndSetSemantics { }) {
    Text("This is hidden")  // Not accessible!
}

// Solution: Add content to parent semantics or remove clear
Box(Modifier.semantics(mergeDescendants = true) { }) {
    Text("This is now accessible")
}
```

**3. Element explicitly hidden**
```kotlin
// Check if invisibleToUser or hideFromAccessibility is set
Modifier.semantics { invisibleToUser() }  // Hides element

// Solution: Remove the hiding modifier
```

**4. Image with null contentDescription**
```kotlin
// Problem: null means decorative/hidden
Image(painter = ..., contentDescription = null)

// Solution: Provide description
Image(painter = ..., contentDescription = "Product photo")
```

## TalkBack Reading Wrong Content

### Symptom
TalkBack announces something different than expected.

### Causes & Solutions

**1. mergeDescendants combining unwanted content**
```kotlin
// Problem: All child text merges
Card(Modifier.semantics(mergeDescendants = true) { }) {
    Text("Price: $10")
    Text("Out of stock")  // Also merged
}

// Solution: Be selective about what merges
Card(Modifier.clickable { }) {
    Text("Price: $10")
    Text(
        "Out of stock",
        modifier = Modifier.clearAndSetSemantics {
            contentDescription = "Currently out of stock"
        }
    )
}
```

**2. contentDescription overriding text**
```kotlin
// Problem: contentDescription replaces text
Text(
    text = "Settings",
    modifier = Modifier.semantics {
        contentDescription = "Configuration"  // Overwrites "Settings"
    }
)

// Solution: Only use contentDescription when needed
Text(text = "Settings")  // Just use the text
```

**3. State description wrong**
```kotlin
// Problem: Stale state
var isExpanded by remember { mutableStateOf(false) }
Modifier.semantics {
    stateDescription = "collapsed"  // Never updates!
}

// Solution: Use current state
Modifier.semantics {
    stateDescription = if (isExpanded) "expanded" else "collapsed"
}
```

## Traversal Order Wrong

### Symptom
TalkBack navigates in an unexpected order.

### Causes & Solutions

**1. Layout order differs from visual order**
```kotlin
// Problem: Row with RTL visual but LTR semantics
Row {
    Spacer(Modifier.weight(1f))
    Text("First visually on right")  // Read first
    Text("Second visually on left")  // Read second
}

// Solution: Use traversalIndex
Row {
    Spacer(Modifier.weight(1f))
    Text(
        "First visually on right",
        modifier = Modifier.semantics { traversalIndex = 1f }
    )
    Text(
        "Second visually on left",
        modifier = Modifier.semantics { traversalIndex = 0f }
    )
}
```

**2. Overlapping elements**
```kotlin
// Problem: FAB over list, read in z-order
Box {
    LazyColumn { ... }
    FloatingActionButton(
        modifier = Modifier.align(Alignment.BottomEnd)
    ) { ... }  // Read after all list items
}

// Solution: Group and order
Box(Modifier.semantics { isTraversalGroup = true }) {
    LazyColumn { ... }
    FloatingActionButton(
        modifier = Modifier
            .align(Alignment.BottomEnd)
            .semantics { traversalIndex = -1f }  // Read before list
    ) { ... }
}
```

## Click Not Working via TalkBack

### Symptom
Double-tap doesn't activate the element.

### Causes & Solutions

**1. onClick on wrong modifier**
```kotlin
// Problem: Nested clickables
Box(Modifier.clickable { outerAction() }) {
    Text(
        "Click me",
        modifier = Modifier.clickable { innerAction() }  // Which one?
    )
}

// Solution: Single clickable at appropriate level
Box(Modifier.clickable { action() }) {
    Text("Click me")
}
```

**2. Custom click handling without semantics**
```kotlin
// Problem: Using pointerInput without semantics
Box(
    Modifier.pointerInput(Unit) {
        detectTapGestures { handleTap() }
    }
)

// Solution: Add semantics action
Box(
    Modifier
        .pointerInput(Unit) { detectTapGestures { handleTap() } }
        .semantics { onClick { handleTap(); true } }
)
```

**3. Action returns false**
```kotlin
// Problem: Action returns false (failure)
Modifier.semantics {
    onClick {
        performClick()
        false  // TalkBack thinks it failed
    }
}

// Solution: Return true on success
Modifier.semantics {
    onClick {
        performClick()
        true
    }
}
```

## Custom Actions Not Showing

### Symptom
Custom actions don't appear in TalkBack menu.

### Causes & Solutions

**1. Empty or null actions list**
```kotlin
// Problem: Conditionally empty
Modifier.semantics {
    customActions = if (canDelete) listOf(...) else emptyList()
}

// Solution: Only set when there are actions
Modifier.semantics {
    if (canDelete) {
        customActions = listOf(
            CustomAccessibilityAction("Delete") { onDelete(); true }
        )
    }
}
```

**2. Actions on merged child**
```kotlin
// Problem: Custom actions on child, not merged parent
Card(Modifier.semantics(mergeDescendants = true) { }) {
    Button(
        modifier = Modifier.semantics {
            customActions = listOf(...)  // Lost in merge!
        }
    ) { }
}

// Solution: Put custom actions on merging parent
Card(
    modifier = Modifier.semantics(mergeDescendants = true) {
        customActions = listOf(
            CustomAccessibilityAction("Delete") { onDelete(); true }
        )
    }
) {
    // Children
}
```

## State Changes Not Announced

### Symptom
When state changes, TalkBack doesn't announce it.

### Causes & Solutions

**1. Missing live region**
```kotlin
// Problem: No live region
Text(text = statusMessage)  // Changes silently

// Solution: Add live region
Text(
    text = statusMessage,
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite
    }
)
```

**2. Wrong live region mode**
```kotlin
// Problem: Polite doesn't interrupt
Modifier.semantics { liveRegion = LiveRegionMode.Polite }

// Solution: Use Assertive for critical updates
Modifier.semantics { liveRegion = LiveRegionMode.Assertive }
```

**3. Component recreated vs updated**
```kotlin
// Problem: Key changes cause recreation, not update
key(statusId) {  // New key = new node = no announcement
    Text(statusMessage, Modifier.semantics { liveRegion = ... })
}

// Solution: Keep same node, update content
Text(statusMessage, Modifier.semantics { liveRegion = ... })
```

## Touch Target Too Small

### Symptom
Accessibility Scanner reports touch target < 48dp.

### Solution
```kotlin
// Problem: Small icon
Icon(
    imageVector = Icons.Default.Close,
    contentDescription = "Close",
    modifier = Modifier.size(24.dp)
)

// Solution 1: Use IconButton (automatically 48dp minimum)
IconButton(onClick = onClose) {
    Icon(Icons.Default.Close, contentDescription = "Close")
}

// Solution 2: Add minimum touch target manually
Icon(
    imageVector = Icons.Default.Close,
    contentDescription = "Close",
    modifier = Modifier
        .size(24.dp)
        .defaultMinSize(minWidth = 48.dp, minHeight = 48.dp)
)
```

## Color Contrast Issues

### Symptom
Accessibility Scanner reports insufficient contrast.

### Solutions
```kotlin
// Check contrast ratios:
// - Normal text: 4.5:1 minimum
// - Large text (18sp+ or 14sp+ bold): 3:1 minimum
// - UI components: 3:1 minimum

// Use Material theme colors (designed for contrast)
Text(
    text = "Readable text",
    color = MaterialTheme.colorScheme.onSurface,
    style = MaterialTheme.typography.bodyLarge
)

// Avoid custom colors without checking contrast
// Use tools like WebAIM Contrast Checker
```

## Test Failures

### Can't Find Node

```kotlin
// Problem: Node doesn't exist or has different semantics
onNodeWithContentDescription("Submit").assertExists()  // Fails

// Debug: Print the tree
onRoot().printToLog("DEBUG")

// Check: Is it merged? Use unmerged tree
onNode(
    hasContentDescription("Submit"),
    useUnmergedTree = true
).assertExists()
```

### Wrong Semantics Values

```kotlin
// Problem: Expected value doesn't match
.assert(SemanticsMatcher.expectValue(SemanticsProperties.Role, Role.Button))

// Debug: Fetch and print the actual value
val node = onNode(hasText("Submit")).fetchSemanticsNode()
println("Actual role: ${node.config.getOrNull(SemanticsProperties.Role)}")
```

## Debugging Tools

### Print Semantics Tree
```kotlin
// In tests
composeTestRule.onRoot().printToLog("SEMANTICS")
composeTestRule.onRoot(useUnmergedTree = true).printToLog("UNMERGED")

// In app (debug builds)
@Composable
fun DebugSemantics(content: @Composable () -> Unit) {
    val owner = LocalSemanticsOwner.current
    LaunchedEffect(Unit) {
        Log.d("Semantics", owner.rootSemanticsNode.toString())
    }
    content()
}
```

### Layout Inspector
1. Open Android Studio Layout Inspector
2. Select your running app
3. Enable "Show accessibility info" in options
4. Click elements to see semantics properties

### Accessibility Scanner
1. Install from Play Store
2. Enable in Settings > Accessibility
3. Run your app
4. Tap the Scanner overlay button
5. Review suggestions

See `references/common_issues.md` for a quick-reference checklist.
