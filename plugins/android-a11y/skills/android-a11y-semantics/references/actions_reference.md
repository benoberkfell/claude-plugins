# Semantics Actions Reference

Complete reference for all Compose semantics actions. All actions return `Boolean` indicating success.

## Click Actions

```kotlin
Modifier.semantics {
    onClick(label = "Open") {
        performClick()
        true
    }

    onLongClick(label = "Show menu") {
        showMenu()
        true
    }
}
```

## Text Actions

```kotlin
Modifier.semantics {
    // Set text content
    setText(label = "Enter text") { newText ->
        onTextChange(newText.text)
        true
    }

    // Set text selection
    setSelection(label = "Select") { start, end, relativeToOriginal ->
        onSelectionChange(start, end)
        true
    }

    // Insert at cursor
    insertTextAtCursor(label = "Insert") { text ->
        insertText(text.text)
        true
    }

    // Clipboard
    copyText(label = "Copy") { copyToClipboard(); true }
    cutText(label = "Cut") { cutToClipboard(); true }
    pasteText(label = "Paste") { pasteFromClipboard(); true }
}
```

## Scroll Actions

```kotlin
Modifier.semantics {
    // Asynchronous scroll
    scrollBy(label = "Scroll") { x, y ->
        scrollState.scrollBy(x, y)
        true
    }

    // Synchronous scroll (preferred)
    scrollByOffset { offset ->
        val consumed = scrollState.scrollByOffset(offset)
        consumed
    }

    // Jump to index
    scrollToIndex(label = "Go to") { index ->
        scrollState.scrollToItem(index)
        true
    }

    // Get viewport size
    getScrollViewportLength(label = "Viewport") {
        viewportSize.toFloat()
    }
}
```

## Progress Actions

```kotlin
Modifier.semantics {
    setProgress(label = "Set value") { newValue ->
        onValueChange(newValue)
        true
    }
}
```

## Expand/Collapse Actions

```kotlin
Modifier.semantics {
    expand(label = "Expand section") {
        setExpanded(true)
        true
    }

    collapse(label = "Collapse section") {
        setExpanded(false)
        true
    }

    dismiss(label = "Dismiss") {
        onDismiss()
        true
    }
}
```

## Focus Actions

```kotlin
Modifier.semantics {
    requestFocus(label = "Focus") {
        focusRequester.requestFocus()
        true
    }
}
```

## Page Navigation

```kotlin
Modifier.semantics {
    pageUp(label = "Previous page") { goToPreviousPage(); true }
    pageDown(label = "Next page") { goToNextPage(); true }
    pageLeft(label = "Page left") { goLeft(); true }
    pageRight(label = "Page right") { goRight(); true }
}
```

## IME Actions

```kotlin
Modifier.semantics {
    onImeAction(
        imeActionType = ImeAction.Search,
        label = "Search"
    ) {
        performSearch()
        true
    }
}
```

IME action types:
- `ImeAction.Default`
- `ImeAction.None`
- `ImeAction.Go`
- `ImeAction.Search`
- `ImeAction.Send`
- `ImeAction.Previous`
- `ImeAction.Next`
- `ImeAction.Done`

## Custom Actions

Custom actions appear in TalkBack's local context menu:

```kotlin
Modifier.semantics {
    customActions = listOf(
        CustomAccessibilityAction("Delete item") {
            deleteItem()
            true
        },
        CustomAccessibilityAction("Share") {
            shareItem()
            true
        },
        CustomAccessibilityAction("Add to favorites") {
            addToFavorites()
            true
        }
    )
}
```

## Text Layout

```kotlin
Modifier.semantics {
    getTextLayoutResult(label = "Layout") { results ->
        results.add(textLayoutResult)
        true
    }
}
```

## Autofill Actions

```kotlin
Modifier.semantics {
    onFillData(label = "Autofill") { fillData ->
        onAutofill(fillData)
        true
    }
}
```

## Text Substitution Actions

```kotlin
Modifier.semantics {
    setTextSubstitution { substitution ->
        textSubstitution = substitution
        true
    }

    showTextSubstitution { show ->
        isShowingSubstitution = show
        true
    }
}
```

## Action Labels

Labels are optional but recommended for better TalkBack announcements:

```kotlin
// Without label: "Double tap to activate"
onClick { performAction(); true }

// With label: "Double tap to open settings"
onClick(label = "open settings") { performAction(); true }
```
