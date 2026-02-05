---
name: android-a11y-patterns
description: Common accessibility patterns for Compose UI. Use when implementing accessible buttons, cards, lists, forms, dialogs, or any interactive components.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Compose Accessibility Patterns

Common patterns for building accessible UI in Jetpack Compose.

## Clickable Elements

### Simple Button

```kotlin
// clickable() automatically adds Button role
Box(
    modifier = Modifier
        .clickable { onButtonClick() }
        .semantics { contentDescription = "Submit form" }
) {
    Icon(Icons.Default.Send, contentDescription = null)
}
```

### Icon Button

```kotlin
IconButton(onClick = { /* action */ }) {
    Icon(
        imageVector = Icons.Default.Menu,
        contentDescription = "Open navigation menu"
    )
}
```

### Decorative vs Actionable Icons

```kotlin
// Decorative (inside button with text) - hide from a11y
Row(Modifier.clickable { }) {
    Icon(Icons.Default.Star, contentDescription = null) // null = decorative
    Text("Favorite")
}

// Actionable (standalone) - must have description
IconButton(onClick = { }) {
    Icon(Icons.Default.Star, contentDescription = "Add to favorites")
}
```

## Cards with Merged Semantics

### Clickable Card

```kotlin
Card(
    modifier = Modifier
        .semantics(mergeDescendants = true) { }
        .clickable { navigateToDetail() }
) {
    Column {
        Text("Article Title")  // Merged
        Text("Author Name")    // Merged
        Text("2 hours ago")    // Merged
    }
}
// TalkBack: "Article Title, Author Name, 2 hours ago, Button, Double tap to activate"
```

### Card with Custom Actions

```kotlin
Card(
    modifier = Modifier
        .semantics(mergeDescendants = true) {
            customActions = listOf(
                CustomAccessibilityAction("Share") { onShare(); true },
                CustomAccessibilityAction("Delete") { onDelete(); true }
            )
        }
        .clickable { navigateToDetail() }
) {
    // Card content
}
```

### Card Overriding Nested Semantics

```kotlin
Card(
    modifier = Modifier
        .clearAndSetSemantics {
            contentDescription = "Event: Concert at 8pm, $50"
            onClick(label = "View details") { navigateToEvent(); true }
        }
) {
    // Complex nested content that would be confusing if read individually
    Row {
        Image(...)
        Column {
            Text("Concert")
            Text("8pm")
            Text("$50")
        }
        IconButton(onClick = { }) { /* This click is now inaccessible */ }
    }
}
```

## Forms

### Text Field with Error

```kotlin
@Composable
fun EmailField(
    email: String,
    error: String?,
    onEmailChange: (String) -> Unit
) {
    TextField(
        value = email,
        onValueChange = onEmailChange,
        label = { Text("Email") },
        isError = error != null,
        modifier = Modifier.semantics {
            if (error != null) {
                error(error)
            }
            contentType = ContentType.EmailAddress
        }
    )
}
```

### Password Field

```kotlin
TextField(
    value = password,
    onValueChange = onPasswordChange,
    label = { Text("Password") },
    visualTransformation = PasswordVisualTransformation(),
    modifier = Modifier.semantics {
        password()
        contentType = ContentType.Password
    }
)
```

### Required Field

```kotlin
TextField(
    value = name,
    onValueChange = onNameChange,
    label = { Text("Name *") },
    modifier = Modifier.semantics {
        // Use stateDescription for required status
        stateDescription = "Required field"
    }
)
```

## Lists

### Accessible LazyColumn

```kotlin
LazyColumn(
    modifier = Modifier.semantics {
        collectionInfo = CollectionInfo(
            rowCount = items.size,
            columnCount = 1
        )
    }
) {
    itemsIndexed(items, key = { _, item -> item.id }) { index, item ->
        ListItem(
            item = item,
            index = index,
            onDelete = { deleteItem(item) }
        )
    }
}

@Composable
fun ListItem(item: Item, index: Int, onDelete: () -> Unit) {
    Row(
        modifier = Modifier
            .semantics(mergeDescendants = true) {
                collectionItemInfo = CollectionItemInfo(
                    rowIndex = index,
                    rowSpan = 1,
                    columnIndex = 0,
                    columnSpan = 1
                )
                customActions = listOf(
                    CustomAccessibilityAction("Delete") { onDelete(); true }
                )
            }
            .clickable { /* navigate */ }
    ) {
        Text(item.title)
        Text(item.subtitle)
    }
}
```

### Selection in Lists

```kotlin
Row(
    modifier = Modifier
        .semantics {
            selected = isSelected
            role = Role.Tab  // or appropriate role
        }
        .clickable { onSelect() }
) {
    Text(item.name)
    if (isSelected) {
        Icon(Icons.Default.Check, contentDescription = null)
    }
}
```

## Toggles

### Switch

```kotlin
Row(
    modifier = Modifier
        .semantics(mergeDescendants = true) { }
        .toggleable(
            value = isEnabled,
            role = Role.Switch,
            onValueChange = onToggle
        )
) {
    Text("Dark mode")
    Switch(
        checked = isEnabled,
        onCheckedChange = null // Handled by parent
    )
}
```

### Checkbox

```kotlin
Row(
    modifier = Modifier
        .semantics(mergeDescendants = true) { }
        .toggleable(
            value = isChecked,
            role = Role.Checkbox,
            onValueChange = onCheckedChange
        )
) {
    Checkbox(
        checked = isChecked,
        onCheckedChange = null
    )
    Text("Accept terms")
}
```

### Radio Group

```kotlin
Column(Modifier.selectableGroup()) {
    options.forEach { option ->
        Row(
            modifier = Modifier
                .semantics { selected = option == selectedOption }
                .selectable(
                    selected = option == selectedOption,
                    role = Role.RadioButton,
                    onClick = { onOptionSelect(option) }
                )
        ) {
            RadioButton(
                selected = option == selectedOption,
                onClick = null
            )
            Text(option.label)
        }
    }
}
```

## Sliders

```kotlin
Column {
    Text("Volume: ${(volume * 100).toInt()}%")
    Slider(
        value = volume,
        onValueChange = onVolumeChange,
        modifier = Modifier.semantics {
            contentDescription = "Volume"
            stateDescription = "${(volume * 100).toInt()} percent"
        }
    )
}
```

## Dialogs

```kotlin
AlertDialog(
    onDismissRequest = onDismiss,
    title = { Text("Delete item?") },
    text = { Text("This action cannot be undone.") },
    confirmButton = {
        TextButton(onClick = onConfirm) {
            Text("Delete")
        }
    },
    dismissButton = {
        TextButton(onClick = onDismiss) {
            Text("Cancel")
        }
    },
    modifier = Modifier.semantics {
        paneTitle = "Delete confirmation dialog"
    }
)
```

## Dynamic Content

### Snackbar/Toast with Live Region

```kotlin
Snackbar(
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite
    }
) {
    Text("Item deleted")
}
```

### Loading State

```kotlin
Box(
    modifier = Modifier.semantics {
        if (isLoading) {
            contentDescription = "Loading"
            progressBarRangeInfo = ProgressBarRangeInfo.Indeterminate
        }
    }
) {
    if (isLoading) {
        CircularProgressIndicator()
    } else {
        Content()
    }
}
```

### Progress with Value

```kotlin
LinearProgressIndicator(
    progress = downloadProgress,
    modifier = Modifier.semantics {
        contentDescription = "Download progress"
        stateDescription = "${(downloadProgress * 100).toInt()} percent complete"
    }
)
```

## Navigation

### Tab Row

```kotlin
TabRow(selectedTabIndex = selectedIndex) {
    tabs.forEachIndexed { index, tab ->
        Tab(
            selected = index == selectedIndex,
            onClick = { onTabSelect(index) },
            modifier = Modifier.semantics {
                selected = index == selectedIndex
                role = Role.Tab
            }
        ) {
            Text(tab.title)
        }
    }
}
```

### Section Headers

```kotlin
Text(
    text = "Settings",
    style = MaterialTheme.typography.titleLarge,
    modifier = Modifier.semantics { heading() }
)
```

## Images

### Content Image

```kotlin
Image(
    painter = painterResource(R.drawable.chart),
    contentDescription = "Sales chart showing 20% growth in Q4"
)
```

### Decorative Image

```kotlin
Image(
    painter = painterResource(R.drawable.background),
    contentDescription = null  // Decorative, hidden from a11y
)
```

### Complex Image

```kotlin
Image(
    painter = painterResource(R.drawable.infographic),
    contentDescription = null,
    modifier = Modifier.semantics {
        contentDescription = "Infographic: 5 steps to success. " +
            "Step 1: Plan. Step 2: Execute. Step 3: Review. " +
            "Step 4: Iterate. Step 5: Ship."
    }
)
```

See `references/patterns_by_component.md` for more patterns organized by component type.
