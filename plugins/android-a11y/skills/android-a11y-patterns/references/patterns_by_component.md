# Accessibility Patterns by Component

Quick reference for accessibility patterns organized by component type.

## Buttons

### Standard Button
```kotlin
Button(onClick = { }) {
    Text("Submit")
}
// Automatically accessible
```

### Icon-Only Button
```kotlin
IconButton(onClick = { }) {
    Icon(
        imageVector = Icons.Default.Delete,
        contentDescription = "Delete item"  // Required!
    )
}
```

### Button with Icon + Text
```kotlin
Button(onClick = { }) {
    Icon(
        Icons.Default.Add,
        contentDescription = null  // Decorative when with text
    )
    Spacer(Modifier.width(8.dp))
    Text("Add item")
}
```

### FAB
```kotlin
FloatingActionButton(onClick = { }) {
    Icon(
        Icons.Default.Add,
        contentDescription = "Create new item"
    )
}
```

## Text Fields

### Basic
```kotlin
TextField(
    value = text,
    onValueChange = onTextChange,
    label = { Text("Username") }  // Provides accessible name
)
```

### With Helper Text
```kotlin
Column {
    TextField(
        value = text,
        onValueChange = onTextChange,
        label = { Text("Password") },
        isError = hasError
    )
    if (hasError) {
        Text(
            "Password must be at least 8 characters",
            color = MaterialTheme.colorScheme.error,
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}
```

### Search Field
```kotlin
TextField(
    value = query,
    onValueChange = onQueryChange,
    placeholder = { Text("Search...") },
    leadingIcon = {
        Icon(Icons.Default.Search, contentDescription = null)
    },
    modifier = Modifier.semantics {
        contentDescription = "Search"
    }
)
```

## Checkboxes

### Standalone (NOT recommended)
```kotlin
// Avoid - checkbox alone is not fully accessible
Checkbox(checked = isChecked, onCheckedChange = onCheckedChange)
```

### With Label (Recommended)
```kotlin
Row(
    Modifier
        .toggleable(
            value = isChecked,
            role = Role.Checkbox,
            onValueChange = onCheckedChange
        )
) {
    Checkbox(checked = isChecked, onCheckedChange = null)
    Text("Remember me", Modifier.padding(start = 8.dp))
}
```

### Tri-State
```kotlin
TriStateCheckbox(
    state = when {
        allSelected -> ToggleableState.On
        noneSelected -> ToggleableState.Off
        else -> ToggleableState.Indeterminate
    },
    onClick = onToggleAll,
    modifier = Modifier.semantics {
        stateDescription = when {
            allSelected -> "All selected"
            noneSelected -> "None selected"
            else -> "Some selected"
        }
    }
)
```

## Radio Buttons

```kotlin
Column(Modifier.selectableGroup()) {
    options.forEach { option ->
        Row(
            Modifier.selectable(
                selected = option == selected,
                role = Role.RadioButton,
                onClick = { onSelect(option) }
            )
        ) {
            RadioButton(selected = option == selected, onClick = null)
            Text(option.label)
        }
    }
}
```

## Switches

```kotlin
Row(
    Modifier.toggleable(
        value = isOn,
        role = Role.Switch,
        onValueChange = onToggle
    )
) {
    Text("Notifications")
    Spacer(Modifier.weight(1f))
    Switch(checked = isOn, onCheckedChange = null)
}
```

## Sliders

### Continuous
```kotlin
Slider(
    value = value,
    onValueChange = onValueChange,
    modifier = Modifier.semantics {
        contentDescription = "Brightness"
    }
)
```

### Stepped
```kotlin
Slider(
    value = value,
    onValueChange = onValueChange,
    steps = 4,  // 5 positions: 0%, 25%, 50%, 75%, 100%
    modifier = Modifier.semantics {
        contentDescription = "Quality level"
        stateDescription = "${(value * 100).toInt()}%"
    }
)
```

## Progress Indicators

### Determinate
```kotlin
LinearProgressIndicator(
    progress = progress,
    modifier = Modifier.semantics {
        progressBarRangeInfo = ProgressBarRangeInfo(
            current = progress,
            range = 0f..1f
        )
    }
)
```

### Indeterminate
```kotlin
CircularProgressIndicator(
    modifier = Modifier.semantics {
        contentDescription = "Loading"
        progressBarRangeInfo = ProgressBarRangeInfo.Indeterminate
    }
)
```

## Cards

### Non-Clickable
```kotlin
Card {
    Column {
        Text("Title")
        Text("Description")
    }
}
// No special handling needed
```

### Clickable
```kotlin
Card(
    onClick = { },
    modifier = Modifier.semantics(mergeDescendants = true) { }
) {
    Column {
        Text("Title")
        Text("Description")
    }
}
```

### With Actions
```kotlin
Card(
    modifier = Modifier.semantics(mergeDescendants = true) {
        customActions = listOf(
            CustomAccessibilityAction("Edit") { onEdit(); true },
            CustomAccessibilityAction("Delete") { onDelete(); true }
        )
    }
) {
    // Content with visible action buttons
    Row {
        Text("Item")
        Spacer(Modifier.weight(1f))
        IconButton(onClick = onEdit) {
            Icon(Icons.Default.Edit, "Edit")  // Still needs description for sighted users
        }
    }
}
```

## Lists

### Simple
```kotlin
LazyColumn {
    items(items) { item ->
        Text(item.name)
    }
}
```

### With Actions per Item
```kotlin
LazyColumn {
    itemsIndexed(items) { index, item ->
        SwipeToDismiss(
            // Swipe gesture not accessible
        ) {
            ListItem(
                headlineContent = { Text(item.name) },
                modifier = Modifier.semantics {
                    // Add accessible alternative to swipe
                    customActions = listOf(
                        CustomAccessibilityAction("Delete") {
                            deleteItem(item)
                            true
                        }
                    )
                }
            )
        }
    }
}
```

## Tabs

```kotlin
TabRow(selectedTabIndex = selectedIndex) {
    tabs.forEachIndexed { index, tab ->
        Tab(
            selected = index == selectedIndex,
            onClick = { onSelect(index) },
            text = { Text(tab.title) }
        )
    }
}
// Tabs handle accessibility automatically with Material
```

## Bottom Navigation

```kotlin
NavigationBar {
    items.forEachIndexed { index, item ->
        NavigationBarItem(
            icon = { Icon(item.icon, contentDescription = null) },
            label = { Text(item.label) },  // Provides accessible name
            selected = index == selectedIndex,
            onClick = { onSelect(index) }
        )
    }
}
```

## Dialogs

### Alert
```kotlin
AlertDialog(
    onDismissRequest = onDismiss,
    title = { Text("Confirm delete") },
    text = { Text("Are you sure?") },
    confirmButton = {
        TextButton(onClick = onConfirm) { Text("Delete") }
    },
    dismissButton = {
        TextButton(onClick = onDismiss) { Text("Cancel") }
    }
)
// Handles focus trapping automatically
```

### Bottom Sheet
```kotlin
ModalBottomSheet(
    onDismissRequest = onDismiss,
    modifier = Modifier.semantics {
        paneTitle = "Options"
    }
) {
    // Content
}
```

## Dropdowns

```kotlin
ExposedDropdownMenuBox(
    expanded = expanded,
    onExpandedChange = { expanded = it }
) {
    TextField(
        value = selectedOption,
        onValueChange = {},
        readOnly = true,
        label = { Text("Country") },
        trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
        modifier = Modifier
            .menuAnchor()
            .semantics { role = Role.DropdownList }
    )
    ExposedDropdownMenu(
        expanded = expanded,
        onDismissRequest = { expanded = false }
    ) {
        options.forEach { option ->
            DropdownMenuItem(
                text = { Text(option) },
                onClick = {
                    onSelect(option)
                    expanded = false
                }
            )
        }
    }
}
```

## Tooltips

```kotlin
TooltipBox(
    positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
    tooltip = {
        PlainTooltip { Text("Copy to clipboard") }
    },
    state = tooltipState
) {
    IconButton(onClick = onCopy) {
        Icon(Icons.Default.ContentCopy, contentDescription = "Copy")
    }
}
```

## Snackbars

```kotlin
Snackbar(
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite
    },
    action = {
        TextButton(onClick = onUndo) { Text("Undo") }
    }
) {
    Text("Item deleted")
}
```
