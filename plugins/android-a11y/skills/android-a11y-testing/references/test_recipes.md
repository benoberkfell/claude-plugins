# Accessibility Test Recipes

Ready-to-use test patterns for common accessibility scenarios.

## Setup

```kotlin
import androidx.compose.ui.test.*
import androidx.compose.ui.test.junit4.createComposeRule
import androidx.compose.ui.semantics.*
import org.junit.Rule
import org.junit.Test

class AccessibilityTest {
    @get:Rule
    val composeTestRule = createComposeRule()
}
```

## Content Description Tests

### Image Has Description
```kotlin
@Test
fun profileImage_hasContentDescription() {
    composeTestRule.setContent {
        ProfileScreen(user = testUser)
    }

    composeTestRule
        .onNode(hasRole(Role.Image) and hasContentDescription("Profile photo"))
        .assertExists()
}
```

### Icon Button Has Description
```kotlin
@Test
fun deleteButton_isAccessible() {
    composeTestRule.setContent {
        DeleteButton(onClick = {})
    }

    composeTestRule
        .onNode(hasContentDescription("Delete") and hasClickAction())
        .assertExists()
        .assertIsEnabled()
}
```

### Decorative Image Hidden
```kotlin
@Test
fun backgroundImage_isNotAccessible() {
    composeTestRule.setContent {
        BackgroundImage()
    }

    // Should not find any image with accessible content
    composeTestRule
        .onAllNodes(hasRole(Role.Image))
        .filter(hasContentDescription())
        .assertCountEquals(0)
}
```

## State Description Tests

### Toggle State Announced
```kotlin
@Test
fun switch_announcesState() {
    var isOn by mutableStateOf(false)

    composeTestRule.setContent {
        SettingsSwitch(
            checked = isOn,
            onCheckedChange = { isOn = it }
        )
    }

    composeTestRule
        .onNode(hasRole(Role.Switch))
        .assertIsOff()

    composeTestRule
        .onNode(hasRole(Role.Switch))
        .performClick()

    composeTestRule
        .onNode(hasRole(Role.Switch))
        .assertIsOn()
}
```

### Selection State
```kotlin
@Test
fun listItem_announcesSelection() {
    composeTestRule.setContent {
        SelectableList(
            items = listOf("A", "B", "C"),
            selectedIndex = 1
        )
    }

    composeTestRule
        .onNode(hasText("B"))
        .assertIsSelected()

    composeTestRule
        .onNode(hasText("A"))
        .assertIsNotSelected()
}
```

### Error State
```kotlin
@Test
fun textField_announcesError() {
    composeTestRule.setContent {
        EmailField(
            value = "invalid",
            error = "Invalid email format"
        )
    }

    val errorValue = composeTestRule
        .onNode(hasText("invalid"))
        .fetchSemanticsNode()
        .config.getOrNull(SemanticsProperties.Error)

    assertThat(errorValue).isEqualTo("Invalid email format")
}
```

## Role Tests

### Button Role
```kotlin
@Test
fun customButton_hasButtonRole() {
    composeTestRule.setContent {
        CustomButton(text = "Click me", onClick = {})
    }

    composeTestRule
        .onNode(hasText("Click me"))
        .assert(SemanticsMatcher.expectValue(SemanticsProperties.Role, Role.Button))
}
```

### Heading Role
```kotlin
@Test
fun sectionTitle_isHeading() {
    composeTestRule.setContent {
        SectionHeader(title = "Settings")
    }

    composeTestRule
        .onNode(hasText("Settings"))
        .assert(SemanticsMatcher.keyIsDefined(SemanticsProperties.Heading))
}
```

## Actions Tests

### Click Action Exists
```kotlin
@Test
fun card_isClickable() {
    composeTestRule.setContent {
        ClickableCard(onClick = {})
    }

    composeTestRule
        .onNode(hasClickAction())
        .assertExists()
}
```

### Custom Actions
```kotlin
@Test
fun listItem_hasSwipeActions() {
    var deleted = false

    composeTestRule.setContent {
        SwipeableListItem(
            text = "Item",
            onDelete = { deleted = true }
        )
    }

    val actions = composeTestRule
        .onNode(hasText("Item"))
        .fetchSemanticsNode()
        .config[SemanticsActions.CustomActions]

    val deleteAction = actions.find { it.label == "Delete" }
    assertThat(deleteAction).isNotNull()

    // Execute the action
    deleteAction?.action?.invoke()
    assertThat(deleted).isTrue()
}
```

### Scroll Actions
```kotlin
@Test
fun lazyColumn_isScrollable() {
    composeTestRule.setContent {
        LongList(itemCount = 100)
    }

    composeTestRule
        .onNode(hasScrollAction())
        .assertExists()
}
```

## Traversal Tests

### Correct Order
```kotlin
@Test
fun form_hasLogicalOrder() {
    composeTestRule.setContent {
        LoginForm()
    }

    val nodes = composeTestRule
        .onAllNodes(hasAnyAncestor(hasTestTag("login_form")))
        .fetchSemanticsNodes()
        .sortedBy { it.config.getOrNull(SemanticsProperties.TraversalIndex) ?: 0f }

    // Verify expected order
    assertThat(nodes[0].config[SemanticsProperties.ContentDescription])
        .contains("Username")
    assertThat(nodes[1].config[SemanticsProperties.ContentDescription])
        .contains("Password")
}
```

### Traversal Groups
```kotlin
@Test
fun card_groupsContent() {
    composeTestRule.setContent {
        ContentCard()
    }

    val cardNode = composeTestRule
        .onNode(hasTestTag("card"))
        .fetchSemanticsNode()

    val isGroup = cardNode.config.getOrNull(SemanticsProperties.IsTraversalGroup)
    assertThat(isGroup).isTrue()
}
```

## Merged Semantics Tests

### Content Merges Correctly
```kotlin
@Test
fun card_mergesAllText() {
    composeTestRule.setContent {
        Card(Modifier.semantics(mergeDescendants = true) { }) {
            Text("Title")
            Text("Description")
            Text("Footer")
        }
    }

    // All text should be in one merged node
    composeTestRule
        .onNode(hasText("Title"))
        .assertTextContains("Description")
        .assertTextContains("Footer")
}
```

### ClearAndSetSemantics
```kotlin
@Test
fun complexComponent_hasSimplifiedSemantics() {
    composeTestRule.setContent {
        ComplexCard(
            modifier = Modifier.clearAndSetSemantics {
                contentDescription = "Simple description"
            }
        )
    }

    // Child content should not be accessible
    composeTestRule
        .onNode(hasText("Internal text"))
        .assertDoesNotExist()

    // Only the cleared semantics visible
    composeTestRule
        .onNode(hasContentDescription("Simple description"))
        .assertExists()
}
```

## Collection Tests

### LazyColumn Collection Info
```kotlin
@Test
fun list_exposesCollectionInfo() {
    composeTestRule.setContent {
        ItemList(items = List(10) { "Item $it" })
    }

    val collectionInfo = composeTestRule
        .onNode(hasScrollAction())
        .fetchSemanticsNode()
        .config[SemanticsProperties.CollectionInfo]

    assertThat(collectionInfo.rowCount).isEqualTo(10)
    assertThat(collectionInfo.columnCount).isEqualTo(1)
}
```

### Item Position
```kotlin
@Test
fun listItem_hasCorrectPosition() {
    composeTestRule.setContent {
        ItemList(items = listOf("First", "Second", "Third"))
    }

    val itemInfo = composeTestRule
        .onNode(hasText("Second"))
        .fetchSemanticsNode()
        .config[SemanticsProperties.CollectionItemInfo]

    assertThat(itemInfo.rowIndex).isEqualTo(1)
}
```

## Live Region Tests

### Snackbar Announces
```kotlin
@Test
fun snackbar_hasLiveRegion() {
    composeTestRule.setContent {
        Snackbar { Text("Message") }
    }

    val liveRegion = composeTestRule
        .onNode(hasText("Message"))
        .fetchSemanticsNode()
        .config.getOrNull(SemanticsProperties.LiveRegion)

    assertThat(liveRegion).isEqualTo(LiveRegionMode.Polite)
}
```

## Touch Target Tests

### Minimum 48dp
```kotlin
@Test
fun smallIcon_meetsMinimumTouchTarget() {
    composeTestRule.setContent {
        SmallIconButton(onClick = {})
    }

    val node = composeTestRule
        .onNode(hasClickAction())
        .fetchSemanticsNode()

    val touchBounds = node.touchBoundsInRoot
    val density = composeTestRule.density

    val widthDp = with(density) { touchBounds.width.toDp() }
    val heightDp = with(density) { touchBounds.height.toDp() }

    // Android minimum touch target is 48dp
    assertThat(widthDp.value).isAtLeast(48f)
    assertThat(heightDp.value).isAtLeast(48f)
}
```

## Progress Tests

### Determinate Progress
```kotlin
@Test
fun progressBar_exposesProgress() {
    composeTestRule.setContent {
        DownloadProgress(progress = 0.75f)
    }

    val progressInfo = composeTestRule
        .onNode(hasProgressBarRangeInfo())
        .fetchSemanticsNode()
        .config[SemanticsProperties.ProgressBarRangeInfo]

    assertThat(progressInfo.current).isEqualTo(0.75f)
    assertThat(progressInfo.range).isEqualTo(0f..1f)
}
```

### Indeterminate Progress
```kotlin
@Test
fun loadingIndicator_isIndeterminate() {
    composeTestRule.setContent {
        LoadingSpinner()
    }

    val progressInfo = composeTestRule
        .onNode(hasProgressBarRangeInfo())
        .fetchSemanticsNode()
        .config[SemanticsProperties.ProgressBarRangeInfo]

    assertThat(progressInfo).isEqualTo(ProgressBarRangeInfo.Indeterminate)
}
```

## Custom Matchers

```kotlin
fun hasRole(role: Role): SemanticsMatcher =
    SemanticsMatcher.expectValue(SemanticsProperties.Role, role)

fun hasContentDescription(): SemanticsMatcher =
    SemanticsMatcher.keyIsDefined(SemanticsProperties.ContentDescription)

fun hasError(error: String): SemanticsMatcher =
    SemanticsMatcher.expectValue(SemanticsProperties.Error, error)

fun hasProgressBarRangeInfo(): SemanticsMatcher =
    SemanticsMatcher.keyIsDefined(SemanticsProperties.ProgressBarRangeInfo)

fun isHeading(): SemanticsMatcher =
    SemanticsMatcher.keyIsDefined(SemanticsProperties.Heading)
```
