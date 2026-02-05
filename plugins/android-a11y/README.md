# Android Accessibility Plugin for Compose

Skills for building accessible Android apps with Jetpack Compose, including hybrid View/Compose architectures.

## Skills Overview

### Foundation Skills

#### 1. android-a11y-semantics
**Use when:** Adding accessibility properties to Compose UI

Covers:
- `Modifier.semantics { }` usage
- Content descriptions, roles, states
- Live regions and headings
- Traversal control
- Actions (click, scroll, progress, custom)

#### 2. android-a11y-patterns
**Use when:** Implementing accessible UI components

Covers:
- Buttons, cards, lists
- Forms and text fields
- Toggles (checkbox, switch, radio)
- Dialogs and navigation
- Images and progress indicators

#### 3. android-a11y-testing
**Use when:** Verifying accessibility or writing a11y tests

Covers:
- Manual TalkBack testing
- Compose UI test matchers and assertions
- Semantics tree debugging
- Automated scanning tools
- Test recipes for common scenarios

### Troubleshooting Skills

#### 4. android-a11y-troubleshooting
**Use when:** Fixing accessibility issues

Covers:
- Element not accessible
- Wrong announcement content
- Traversal order problems
- Actions not working
- State not announced
- Touch target size issues
- View-based announcements (AccessibilityDelegate, live regions)

### Advanced Skills

#### 5. android-a11y-focus-architecture
**Use when:** Understanding how focus works, debugging desynchronization

Covers:
- Dual-focus model (Input Focus vs Accessibility Focus)
- Accessibility Node Tree construction
- How TalkBack interacts with apps
- Initial focus on screen entry
- Focus search algorithm

#### 6. android-a11y-scrollable-a11y
**Use when:** Fixing issues in scrollable containers

Covers:
- RecyclerView scroll blocking (`importantForAccessibility`)
- Focus traps in carousels (`descendantFocusability`)
- HorizontalPager/ViewPager2 focus jump
- LazyColumn/LazyRow issues
- ComposeView in RecyclerView

#### 7. android-a11y-focus-restoration
**Use when:** Preserving focus across navigation

Covers:
- Fragment focus preservation
- Compose Navigation focus restoration
- FocusRequester + rememberSaveable patterns
- Configuration change handling
- Initial focus on screen entry

#### 8. android-a11y-hybrid-debugging
**Use when:** Debugging View/Compose interop issues

Covers:
- ComposeView in RecyclerView focus stealing
- WebView accessibility integration
- ADB debugging (dumpsys input, dumpsys accessibility)
- TreeDebug via logcat
- Layout Inspector semantics analysis

## Decision Tree

```
Need to make UI accessible?
├── Adding semantics to composables → android-a11y-semantics
├── Building accessible components → android-a11y-patterns
├── Testing accessibility → android-a11y-testing
└── Fixing a11y bugs?
    ├── Basic issues (missing text, wrong order) → android-a11y-troubleshooting
    ├── Focus-related issues?
    │   ├── Understanding how focus works → android-a11y-focus-architecture
    │   ├── Focus lost after navigation → android-a11y-focus-restoration
    │   └── Focus issues in lists/pagers → android-a11y-scrollable-a11y
    └── Hybrid View/Compose app → android-a11y-hybrid-debugging
```

## Quick Reference

| Task | Skill |
|------|-------|
| Add contentDescription | semantics |
| Set role (Button, Checkbox, etc.) | semantics |
| Make card merge child content | patterns |
| Add custom TalkBack actions | semantics |
| Test with TalkBack | testing |
| Write a11y unit tests | testing |
| Debug semantics tree | testing/troubleshooting |
| Fix traversal order | troubleshooting |
| Fix missing announcements | troubleshooting |
| Understand input vs a11y focus | focus-architecture |
| RecyclerView can't scroll with TalkBack | scrollable-a11y |
| HorizontalPager focus jumps | scrollable-a11y |
| Focus lost on back navigation | focus-restoration |
| Initial focus on screen entry | focus-restoration |
| ComposeView steals focus | hybrid-debugging |
| WebView not accessible | hybrid-debugging |
| ADB accessibility debugging | hybrid-debugging |

## Skill Connections

```
                  ┌─────────────────────────┐
                  │ android-a11y-semantics  │ (Foundation)
                  └───────────┬─────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│android-a11y-    │ │android-a11y-    │ │android-a11y-focus-  │
│patterns         │ │testing          │ │architecture         │
└────────┬────────┘ └────────┬────────┘ └──────────┬──────────┘
         │                   │                     │
         ▼                   ▼          ┌─────────┴─────────┐
┌─────────────────┐ ┌─────────────────────┐ ┌───────────────────┐
│scrollable-a11y  │ │hybrid-debugging     │ │focus-restoration  │
└────────┬────────┘ └──────────┬──────────┘ └───────────────────┘
         │                     │
         └─────────────────────┘
                   │
                   ▼
         ┌─────────────────────────┐
         │android-a11y-            │
         │troubleshooting          │
         └─────────────────────────┘
```

## Based On

This plugin is built from analysis of:
- Official Jetpack Compose UI semantics implementation in the AndroidX repository
- TalkBack advanced troubleshooting patterns
- View/Compose interoperability best practices
