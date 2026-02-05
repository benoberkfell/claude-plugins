# Semantics Properties Reference

Complete reference for all Compose semantics properties.

## Content Properties

| Property | Type | Description |
|----------|------|-------------|
| `contentDescription` | `String` | Description for elements without visible text |
| `text` | `AnnotatedString` | The text content (set automatically by Text composables) |
| `stateDescription` | `String` | Current state description |
| `editableText` | `AnnotatedString` | Text field content shown to a11y services |
| `inputText` | `AnnotatedString` | Raw text input after transformations |

### Usage

```kotlin
Modifier.semantics {
    contentDescription = "Profile photo of John Doe"
    stateDescription = "Playing"
    text = AnnotatedString("Button label")
}
```

## Structure Properties

| Property | Type | Description |
|----------|------|-------------|
| `heading()` | Unit | Marks element as a heading |
| `paneTitle` | `String` | Title for distinct visual panes |
| `collectionInfo` | `CollectionInfo` | Collection structure (row/column counts) |
| `collectionItemInfo` | `CollectionItemInfo` | Item position in collection |
| `selectableGroup()` | Unit | Marks group of selectable items |

### Collection Usage

```kotlin
// Parent
Modifier.semantics {
    collectionInfo = CollectionInfo(
        rowCount = 10,      // -1 if unknown
        columnCount = 2
    )
}

// Item
Modifier.semantics {
    collectionItemInfo = CollectionItemInfo(
        rowIndex = 0,
        rowSpan = 1,
        columnIndex = 0,
        columnSpan = 1
    )
}
```

## State Properties

| Property | Type | Description |
|----------|------|-------------|
| `selected` | `Boolean` | Whether item is selected |
| `toggleableState` | `ToggleableState` | On/Off/Indeterminate |
| `focused` | `Boolean` | Whether element has focus |
| `disabled()` | Unit | Marks element as disabled |
| `password()` | Unit | Marks as password field |
| `isEditable` | `Boolean` | Whether text is editable |
| `error` | `String` | Error message for invalid input |

### Toggle State Values

```kotlin
ToggleableState.On           // Checked
ToggleableState.Off          // Unchecked
ToggleableState.Indeterminate // Partially checked
```

## Role Property

```kotlin
Modifier.semantics {
    role = Role.Button
    role = Role.Checkbox
    role = Role.Switch
    role = Role.RadioButton
    role = Role.Tab
    role = Role.Image
    role = Role.DropdownList
}
```

## Scroll Properties

| Property | Type | Description |
|----------|------|-------------|
| `horizontalScrollAxisRange` | `ScrollAxisRange` | Horizontal scroll info |
| `verticalScrollAxisRange` | `ScrollAxisRange` | Vertical scroll info |

```kotlin
Modifier.semantics {
    verticalScrollAxisRange = ScrollAxisRange(
        value = { scrollState.value.toFloat() },
        maxValue = { scrollState.maxValue.toFloat() },
        reverseScrolling = false
    )
}
```

## Progress Properties

```kotlin
Modifier.semantics {
    progressBarRangeInfo = ProgressBarRangeInfo(
        current = 0.5f,
        range = 0f..1f,
        steps = 0  // 0 = continuous
    )
}

// Indeterminate progress
Modifier.semantics {
    progressBarRangeInfo = ProgressBarRangeInfo.Indeterminate
}
```

## Live Region Property

```kotlin
Modifier.semantics {
    liveRegion = LiveRegionMode.Polite    // Wait for current speech
    liveRegion = LiveRegionMode.Assertive // Interrupt immediately
}
```

## Traversal Properties

| Property | Type | Description |
|----------|------|-------------|
| `isTraversalGroup` | `Boolean` | Group children for traversal |
| `traversalIndex` | `Float` | Order within group (default 0f) |

```kotlin
Modifier.semantics {
    isTraversalGroup = true
    traversalIndex = -1f  // Before default
}
```

## Visibility Properties

| Property | Description |
|----------|-------------|
| `invisibleToUser()` | Hide from accessibility (deprecated) |
| `hideFromAccessibility()` | Hide from accessibility services |

## Testing Properties

| Property | Type | Description |
|----------|------|-------------|
| `testTag` | `String` | Tag for UI testing (not exposed to a11y) |
| `testTagsAsResourceId` | `Boolean` | Expose testTags as resource-id |

## Autofill Properties

| Property | Type | Description |
|----------|------|-------------|
| `contentType` | `ContentType` | Field type for autofill |
| `contentDataType` | `ContentDataType` | Data type suggestion |

```kotlin
Modifier.semantics {
    contentType = ContentType.EmailAddress
    contentType = ContentType.Password
    contentType = ContentType.Username
}
```

## Text Substitution Properties

For privacy-sensitive data:

```kotlin
Modifier.semantics {
    textSubstitution = AnnotatedString("***")
    isShowingTextSubstitution = true
}
```
