---
name: android-a11y-testing
description: Test Compose accessibility with TalkBack, automated scanners, and UI tests. Use when verifying accessibility, writing a11y tests, or debugging TalkBack issues.
allowed-tools: Read, Glob, Grep, Edit, Write, Bash
---

# Compose Accessibility Testing

How to verify your Compose UI is accessible through manual testing, automated tools, and unit tests.

## Manual Testing with TalkBack

### Enable TalkBack
1. Settings > Accessibility > TalkBack > On
2. Or use shortcut: Hold both volume keys for 3 seconds

### TalkBack Gestures
- **Swipe right/left**: Move to next/previous element
- **Double tap**: Activate (click)
- **Double tap and hold**: Long press
- **Swipe up then right**: Open TalkBack menu (custom actions)
- **Two-finger swipe**: Scroll

### What to Check

1. **All interactive elements reachable** - Can you navigate to every button, link, field?
2. **Meaningful descriptions** - Does TalkBack announce what the element is?
3. **State announced** - Is selected/checked/expanded state clear?
4. **Logical order** - Does navigation order make sense?
5. **No redundancy** - Are descriptions duplicated unnecessarily?
6. **Actions work** - Can you activate buttons, toggle switches via TalkBack?

## Automated Scanning

### Accessibility Scanner App
1. Install from Play Store: "Accessibility Scanner" by Google
2. Enable in Accessibility settings
3. Open your app
4. Tap the Scanner FAB to analyze current screen
5. Review suggestions for touch target size, contrast, labels

### Compose UI Testing with Semantics

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun button_hasContentDescription() {
    composeTestRule.setContent {
        MyScreen()
    }

    composeTestRule
        .onNodeWithContentDescription("Submit")
        .assertExists()
}

@Test
fun checkbox_hasCorrectRole() {
    composeTestRule.setContent {
        MyCheckbox(checked = false, onCheckedChange = {})
    }

    composeTestRule
        .onNode(hasRole(Role.Checkbox))
        .assertExists()
}
```

## Semantics Test Matchers

### Finding Nodes

```kotlin
// By content description
onNodeWithContentDescription("Search")
onAllNodesWithContentDescription("Star")

// By text
onNodeWithText("Submit")
onAllNodesWithText("Item")

// By test tag (not exposed to a11y, for testing only)
onNodeWithTag("submit_button")

// By role
onNode(hasRole(Role.Button))
onNode(hasRole(Role.Checkbox))

// By state
onNode(isSelected())
onNode(isNotSelected())
onNode(isToggleable())
onNode(isFocused())

// Combining matchers
onNode(hasText("Accept") and hasRole(Role.Checkbox))
onNode(hasContentDescription("Delete") and isEnabled())
```

### Assertions

```kotlin
// Existence
.assertExists()
.assertDoesNotExist()

// State
.assertIsDisplayed()
.assertIsEnabled()
.assertIsNotEnabled()
.assertIsSelected()
.assertIsNotSelected()
.assertIsFocused()
.assertIsOn()  // For toggles
.assertIsOff()

// Content
.assertTextEquals("Expected text")
.assertTextContains("partial")
.assertContentDescriptionEquals("Description")
.assertContentDescriptionContains("partial")

// Semantics values
.assert(SemanticsMatcher.expectValue(SemanticsProperties.Role, Role.Button))
.assert(SemanticsMatcher.expectValue(SemanticsProperties.Selected, true))
```

### Actions in Tests

```kotlin
// Click
.performClick()

// Text input
.performTextInput("Hello")
.performTextClearance()
.performTextReplacement("New text")

// Scroll
.performScrollTo()
.performScrollToIndex(5)

// Semantic actions
.performSemanticsAction(SemanticsActions.OnClick)
.performSemanticsAction(SemanticsActions.SetProgress) { it(0.5f) }
```

## Testing Custom Semantics

```kotlin
@Test
fun slider_exposesProgressInfo() {
    composeTestRule.setContent {
        VolumeSlider(volume = 0.5f, onVolumeChange = {})
    }

    val progressInfo = composeTestRule
        .onNode(hasContentDescription("Volume"))
        .fetchSemanticsNode()
        .config[SemanticsProperties.ProgressBarRangeInfo]

    assertThat(progressInfo.current).isEqualTo(0.5f)
    assertThat(progressInfo.range).isEqualTo(0f..1f)
}

@Test
fun card_hasCustomActions() {
    composeTestRule.setContent {
        ItemCard(item = testItem, onDelete = {}, onShare = {})
    }

    val customActions = composeTestRule
        .onNode(hasText("Test Item"))
        .fetchSemanticsNode()
        .config[SemanticsActions.CustomActions]

    assertThat(customActions.map { it.label }).containsExactly("Delete", "Share")
}
```

## Testing Merged Semantics

```kotlin
@Test
fun card_mergesChildText() {
    composeTestRule.setContent {
        Card(Modifier.semantics(mergeDescendants = true) { }) {
            Column {
                Text("Title")
                Text("Subtitle")
            }
        }
    }

    // Both texts should be in the same node
    composeTestRule
        .onNode(hasText("Title") and hasText("Subtitle"))
        .assertExists()
}
```

## Debugging Semantics Tree

### Print Semantics Tree

```kotlin
@Test
fun debug_printTree() {
    composeTestRule.setContent {
        MyScreen()
    }

    // Print merged tree (what TalkBack sees)
    composeTestRule.onRoot().printToLog("SEMANTICS")

    // Print unmerged tree (all nodes)
    composeTestRule.onRoot(useUnmergedTree = true).printToLog("UNMERGED")
}
```

### Sample Output

```
SEMANTICS:
Printing with useUnmergedTree = 'false'
Node #1 at (l=0, t=0, r=1080, b=1920)
 |-Node #2 at (l=0, t=0, r=1080, b=200)
 |   Text = 'Title'
 |   Role = 'Heading'
 |-Node #3 at (l=0, t=200, r=1080, b=400)
 |   Text = 'Content, Subtitle'
 |   Role = 'Button'
 |   Actions = [OnClick]
```

## Integration Testing

### Espresso Accessibility Checks

Enable automatic accessibility checks in Espresso:

```kotlin
@RunWith(AndroidJUnit4::class)
class AccessibilityTest {

    init {
        AccessibilityChecks.enable()
            .setRunChecksFromRootView(true)
    }

    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Test
    fun screenPassesAccessibilityChecks() {
        // Any Espresso action will trigger a11y checks
        onView(withId(R.id.button)).perform(click())
    }
}
```

### ATF Integration

Use Accessibility Test Framework directly:

```kotlin
@Test
fun checkAccessibility() {
    val accessibilityValidator = AccessibilityValidator()

    composeTestRule.setContent {
        MyScreen()
    }

    val rootView = composeTestRule.activity.window.decorView
    val results = accessibilityValidator.checkHierarchy(rootView)

    val errors = results.filter { it.type == AccessibilityCheckResult.Type.ERROR }
    assertThat(errors).isEmpty()
}
```

## Common Test Scenarios

### Test Touch Target Size

```kotlin
@Test
fun iconButton_meetsTouchTargetSize() {
    composeTestRule.setContent {
        IconButton(onClick = {}) {
            Icon(Icons.Default.Add, contentDescription = "Add")
        }
    }

    val bounds = composeTestRule
        .onNode(hasContentDescription("Add"))
        .fetchSemanticsNode()
        .touchBoundsInRoot

    val widthDp = with(composeTestRule.density) { bounds.width.toDp() }
    val heightDp = with(composeTestRule.density) { bounds.height.toDp() }

    assertThat(widthDp).isAtLeast(48.dp)
    assertThat(heightDp).isAtLeast(48.dp)
}
```

### Test Live Regions

```kotlin
@Test
fun snackbar_announcesChanges() {
    composeTestRule.setContent {
        MySnackbar(message = "Item deleted")
    }

    composeTestRule
        .onNode(hasText("Item deleted"))
        .assert(SemanticsMatcher.expectValue(
            SemanticsProperties.LiveRegion,
            LiveRegionMode.Polite
        ))
}
```

### Test Collection Info

```kotlin
@Test
fun list_hasCollectionInfo() {
    val items = listOf("A", "B", "C")

    composeTestRule.setContent {
        LazyColumn(Modifier.semantics {
            collectionInfo = CollectionInfo(items.size, 1)
        }) {
            items(items) { Text(it) }
        }
    }

    val collectionInfo = composeTestRule
        .onNode(hasScrollAction())
        .fetchSemanticsNode()
        .config[SemanticsProperties.CollectionInfo]

    assertThat(collectionInfo.rowCount).isEqualTo(3)
}
```

See `references/test_recipes.md` for more test examples.
