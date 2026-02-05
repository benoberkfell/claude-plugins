# Common Accessibility Issues Checklist

Quick reference for diagnosing accessibility problems.

## Element Not Accessible

| Check | Fix |
|-------|-----|
| Image has `contentDescription = null` | Add description |
| Canvas without semantics | Add `Modifier.semantics { contentDescription = "..." }` |
| Parent uses `clearAndSetSemantics` | Move element outside or add to parent's semantics |
| `invisibleToUser()` applied | Remove if element should be accessible |
| Element outside bounds | Check layout, may be clipped |

## Wrong Announcement

| Check | Fix |
|-------|-----|
| `mergeDescendants = true` on parent | Use `clearAndSetSemantics` for precise control |
| `contentDescription` overriding text | Only use contentDescription when necessary |
| Stale state in semantics lambda | Use current state values |
| Multiple text nodes merging | Control with traversal groups |

## Wrong Order

| Check | Fix |
|-------|-----|
| Visual order ≠ layout order | Use `traversalIndex` |
| Overlapping elements | Group with `isTraversalGroup` |
| Absolute positioning | Set explicit `traversalIndex` |
| Dialog/popup order | Check pane titles and focus |

## Click Doesn't Work

| Check | Fix |
|-------|-----|
| Nested clickables | Remove inner clickable |
| `pointerInput` without semantics | Add `onClick` action |
| Action returns `false` | Return `true` on success |
| Element disabled | Check `disabled()` modifier |

## Custom Actions Missing

| Check | Fix |
|-------|-----|
| Actions on merged child | Move to merging parent |
| Conditional empty list | Only set when non-empty |
| Actions after clear | Set actions in `clearAndSetSemantics` |

## State Not Announced

| Check | Fix |
|-------|-----|
| Missing live region | Add `liveRegion = LiveRegionMode.Polite` |
| Need immediate announcement | Use `LiveRegionMode.Assertive` |
| Node recreated on change | Keep same composable, update state |

## Touch Target Too Small

| Check | Fix |
|-------|-----|
| Custom clickable < 48dp | Use `defaultMinSize(48.dp, 48.dp)` |
| Icon without button wrapper | Wrap in `IconButton` |
| Dense layouts | Consider spacing for accessibility |

## Test Failures

| Issue | Debug Step |
|-------|------------|
| Node not found | Print tree with `printToLog()` |
| Merged vs unmerged | Try `useUnmergedTree = true` |
| Wrong value | Fetch node and print config |
| Timing issue | Add `waitForIdle()` |

## Quick Diagnostics

### TalkBack Not Working
1. Is TalkBack on? (Settings > Accessibility)
2. Is element on screen? (Check bounds)
3. Has semantics? (Check for semantics modifier)
4. Parent clearing semantics? (Check ancestors)

### Wrong Content Read
1. Print semantics tree
2. Check merge behavior
3. Verify contentDescription vs text
4. Check state values

### Can't Navigate
1. Element has no actions?
2. Hidden from accessibility?
3. Inside cleared semantics?
4. Touch target too small?

## Semantics Tree Debug

```kotlin
// Print merged tree (what TalkBack sees)
composeTestRule.onRoot().printToLog("MERGED")

// Print unmerged tree (all nodes)
composeTestRule.onRoot(useUnmergedTree = true).printToLog("UNMERGED")
```

## Common Modifier Issues

### Order Matters
```kotlin
// These are different!
Modifier
    .semantics { contentDescription = "A" }
    .semantics { contentDescription = "B" }  // "A" wins (outer)

Modifier
    .clickable { }
    .semantics { role = Role.Tab }  // Role may be overridden by clickable
```

### Merging Behavior
```kotlin
// Outer semantics{ } always wins for same property
// mergeDescendants pulls child properties up
// clearAndSetSemantics removes all descendants
```

## Material Component Defaults

| Component | Default Role | Default Actions |
|-----------|-------------|-----------------|
| Button | Button | onClick |
| IconButton | Button | onClick |
| Checkbox | Checkbox | onClick, toggle |
| Switch | Switch | onClick, toggle |
| RadioButton | RadioButton | onClick |
| Slider | - | setProgress |
| TextField | - | setText, setSelection |
| Tab | Tab | onClick |
| Card | - | (none unless clickable) |

## Accessibility Scanner Warnings

| Warning | Common Fix |
|---------|------------|
| Touch target size | Use 48x48dp minimum |
| Color contrast | Use Material theme colors |
| Missing label | Add contentDescription |
| Duplicate descriptions | Make descriptions unique |
| Speakable text | Avoid technical jargon |
