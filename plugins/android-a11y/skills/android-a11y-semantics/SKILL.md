---
name: android-a11y-semantics
description: Use Compose semantics properties and actions to make UI accessible. Use when adding contentDescription, roles, state descriptions, live regions, or any semantics modifiers to Compose components.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Compose Accessibility Semantics

This skill covers the semantics system in Jetpack Compose - the foundation for accessibility.

## Core Concept

Compose uses a **semantics tree** to expose UI information to accessibility services like TalkBack. Every accessible element needs semantic properties that describe what it is and what it does.

## The Semantics Modifier

```kotlin
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.semantics.*

// Basic usage
Modifier.semantics {
    contentDescription = "Profile picture"
}

// Merge descendants into single node
Modifier.semantics(mergeDescendants = true) {
    // All child semantics merge into this node
}

// Clear and replace all descendant semantics
Modifier.clearAndSetSemantics {
    contentDescription = "Complete card"
    onClick { navigateToDetails(); true }
}
```

## Essential Properties

### Content Description
For elements without text (images, icons, decorative elements):

```kotlin
Image(
    painter = painterResource(R.drawable.profile),
    contentDescription = null, // Decorative, hidden from a11y
    modifier = Modifier.semantics {
        contentDescription = "User profile photo" // Override if needed
    }
)

Icon(
    imageVector = Icons.Default.Add,
    contentDescription = "Add item" // Required for actionable icons
)
```

### Role
Tells TalkBack what type of element this is:

```kotlin
Modifier.semantics {
    role = Role.Button      // Announced as "button"
    role = Role.Checkbox    // "checkbox"
    role = Role.Switch      // "switch"
    role = Role.RadioButton // "radio button"
    role = Role.Tab         // "tab"
    role = Role.Image       // "image"
    role = Role.DropdownList // "drop down list"
}
```

### State Description
Current state for stateful elements:

```kotlin
Modifier.semantics {
    stateDescription = if (isExpanded) "expanded" else "collapsed"
}

// For toggles, use toggleableState instead
Modifier.semantics {
    toggleableState = if (isChecked) ToggleableState.On else ToggleableState.Off
}
```

### Live Regions
Announce changes automatically:

```kotlin
// For snackbars, toasts, dynamic content
Modifier.semantics {
    liveRegion = LiveRegionMode.Polite    // Announce after current speech
    liveRegion = LiveRegionMode.Assertive // Interrupt to announce
}
```

### Heading
Mark section headers for navigation:

```kotlin
Text(
    text = "Settings",
    modifier = Modifier.semantics { heading() }
)
```

## Actions

Actions let users interact via accessibility services:

```kotlin
Modifier.semantics {
    // Click action
    onClick(label = "Open details") {
        navigateToDetails()
        true // Return true on success
    }

    // Long click
    onLongClick(label = "Show options") {
        showContextMenu()
        true
    }

    // Custom actions (shown in TalkBack menu)
    customActions = listOf(
        CustomAccessibilityAction("Delete") { onDelete(); true },
        CustomAccessibilityAction("Share") { onShare(); true }
    )
}
```

### Slider/Progress Actions

```kotlin
Modifier.semantics {
    progressBarRangeInfo = ProgressBarRangeInfo(
        current = volume,
        range = 0f..100f,
        steps = 10
    )
    setProgress(label = "Set volume") { newValue ->
        onVolumeChange(newValue)
        true
    }
}
```

### Text Field Actions

```kotlin
Modifier.semantics {
    editableText = AnnotatedString(textValue)
    setText { newText ->
        onTextChange(newText.text)
        true
    }
    setSelection { start, end, _ ->
        onSelectionChange(start, end)
        true
    }
}
```

## Traversal Control

Control TalkBack's navigation order:

```kotlin
// Group related elements for sequential traversal
Column(Modifier.semantics { isTraversalGroup = true }) {
    // Children traversed together before moving to next group
}

// Fine-tune order within groups
Modifier.semantics {
    traversalIndex = -1f  // Earlier (negative = before default)
    traversalIndex = 0f   // Default
    traversalIndex = 1f   // Later (positive = after default)
}
```

## Collections

For lists and grids:

```kotlin
// Parent container
LazyColumn(
    modifier = Modifier.semantics {
        collectionInfo = CollectionInfo(
            rowCount = items.size,  // Use -1 if unknown
            columnCount = 1
        )
    }
)

// Each item
Row(
    modifier = Modifier.semantics {
        collectionItemInfo = CollectionItemInfo(
            rowIndex = index,
            rowSpan = 1,
            columnIndex = 0,
            columnSpan = 1
        )
    }
)
```

## Hiding from Accessibility

```kotlin
// Hide decorative elements
Modifier.semantics {
    invisibleToUser() // Deprecated but still works
}

// Better approach - use clearAndSetSemantics on parent
Modifier.clearAndSetSemantics {
    contentDescription = "Meaningful description"
}
```

## Property Reference

See `references/properties_reference.md` for the complete list of all semantics properties and their usage.

## Action Reference

See `references/actions_reference.md` for the complete list of all semantics actions.
