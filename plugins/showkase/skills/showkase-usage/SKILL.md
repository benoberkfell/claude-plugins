---
name: showkase-usage
description: Use Showkase features to annotate and document Compose UI elements. Use when adding @ShowkaseComposable, @ShowkaseColor, @ShowkaseTypography annotations, creating component previews, documenting design system elements, setting up component styles, or working with the Showkase browser.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Using Showkase

Guide for annotating and documenting Jetpack Compose UI elements with Showkase.

## Core Annotations

### @ShowkaseComposable - For Composable Functions

Annotate composable functions to include them in the Showkase browser:

```kotlin
import com.airbnb.android.showkase.annotation.ShowkaseComposable

@ShowkaseComposable(name = "Primary Button", group = "Buttons")
@Composable
fun PrimaryButtonPreview() {
    PrimaryButton(text = "Click me", onClick = {})
}
```

**Or use the standard @Preview annotation** (Showkase has first-class support):

```kotlin
import androidx.compose.ui.tooling.preview.Preview

@Preview(name = "Primary Button", group = "Buttons")
@Composable
fun PrimaryButtonPreview() {
    PrimaryButton(text = "Click me", onClick = {})
}
```

#### Properties

| Property | Description |
|----------|-------------|
| `name` | Display name (defaults to function name) |
| `group` | Grouping category (defaults to enclosing class or "Default Group") |
| `widthDp` | Constrain preview width |
| `heightDp` | Constrain preview height |
| `skip` | Set `true` to exclude from browser |
| `styleName` | Name of style variant |
| `defaultStyle` | Set `true` if this is the default/primary style |

#### Requirements for Composables

Functions must either:
- Have no parameters, OR
- Use `@PreviewParameter` with a `PreviewParameterProvider`

```kotlin
// Using PreviewParameter
data class User(val name: String, val avatar: String)

class UserProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User("Alice", "avatar1.png"),
        User("Bob", "avatar2.png")
    )
}

@Preview(name = "User Card", group = "Cards")
@Composable
fun UserCardPreview(
    @PreviewParameter(UserProvider::class) user: User
) {
    UserCard(user = user)
}
// Creates 2 previews - one for each User
```

### @ShowkaseColor - For Color Properties

Document color values in your design system:

```kotlin
import com.airbnb.android.showkase.annotation.ShowkaseColor
import androidx.compose.ui.graphics.Color

@ShowkaseColor(name = "Primary", group = "Brand Colors")
val primaryColor = Color(0xFF6200EE)

@ShowkaseColor(name = "Secondary", group = "Brand Colors")
val secondaryColor = Color(0xFF03DAC6)

@ShowkaseColor(name = "Error", group = "Semantic Colors")
val errorColor = Color(0xFFB00020)
```

### @ShowkaseTypography - For TextStyle Properties

Document typography styles:

```kotlin
import com.airbnb.android.showkase.annotation.ShowkaseTypography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

@ShowkaseTypography(name = "Heading 1", group = "Headlines")
val h1 = TextStyle(
    fontWeight = FontWeight.Bold,
    fontSize = 32.sp,
    letterSpacing = 0.sp
)

@ShowkaseTypography(name = "Body", group = "Body Text")
val body1 = TextStyle(
    fontWeight = FontWeight.Normal,
    fontSize = 16.sp,
    lineHeight = 24.sp
)
```

## Component Style Variants

Document multiple styles of a single component using `styleName` and `defaultStyle`:

```kotlin
// Default style
@ShowkaseComposable(
    name = "Button",
    group = "Buttons",
    defaultStyle = true
)
@Composable
fun ButtonDefaultPreview() {
    Button(size = ButtonSize.Medium)
}

// Style variants
@ShowkaseComposable(
    name = "Button",
    group = "Buttons",
    styleName = "Large"
)
@Composable
fun ButtonLargePreview() {
    Button(size = ButtonSize.Large)
}

@ShowkaseComposable(
    name = "Button",
    group = "Buttons",
    styleName = "Small"
)
@Composable
fun ButtonSmallPreview() {
    Button(size = ButtonSize.Small)
}
```

All styles appear together on the same screen in the browser.

## Custom/Multi-Preview Annotations

Create reusable preview configurations:

```kotlin
// Define custom preview annotation
@Preview(name = "Light Mode", group = "Theme")
@Preview(name = "Dark Mode", group = "Theme", uiMode = Configuration.UI_MODE_NIGHT_YES)
annotation class ThemePreview

// Use it
@ThemePreview
@Composable
fun MyComponentPreview() {
    MyComponent()
}
// Creates 2 previews automatically
```

**Note:** Stacked `@Preview` annotations on custom annotations require KSP. KAPT only supports single `@Preview` per custom annotation.

For KAPT, register custom annotations:
```kotlin
kapt {
    arguments {
        arg("multiPreviewType", "com.yourpackage.ThemePreview")
    }
}
```

## Constrained Previews

Set specific dimensions for previews:

```kotlin
@ShowkaseComposable(
    name = "Card",
    group = "Cards",
    widthDp = 300,
    heightDp = 200
)
@Composable
fun CardPreview() {
    Card(title = "Example", content = "Content here")
}
```

## Accessing Showkase Metadata

Get programmatic access to all documented UI elements:

```kotlin
import com.airbnb.android.showkase.models.Showkase

// Get all metadata
val metadata = Showkase.getMetadata()

// Access components
val components = metadata.componentList
components.forEach { component ->
    println("${component.group} / ${component.componentName}")
}

// Access colors
val colors = metadata.colorList
colors.forEach { color ->
    println("${color.colorGroup} / ${color.colorName}: ${color.color}")
}

// Access typography
val typography = metadata.typographyList
typography.forEach { style ->
    println("${style.typographyGroup} / ${style.typographyName}")
}
```

Useful for:
- Screenshot testing
- Custom documentation generators
- Design system validation

## Auto-Generated Permutations

Showkase automatically creates 5 permutations for each composable:
1. **Basic** - Default rendering
2. **Dark Mode** - Dark theme applied
3. **RTL** - Right-to-left layout
4. **Font Scaled** - Larger font size
5. **Display Scaled** - Larger display density

## Best Practices

### Organization
- Use consistent `group` names across your codebase
- Group by component type: "Buttons", "Cards", "Inputs", "Typography"
- Or by feature: "Authentication", "Profile", "Settings"

### Naming
- Use descriptive `name` values that explain the variant
- Include state in names: "Button - Disabled", "TextField - Error"

### Preview Functions
- Keep preview functions simple and focused
- Create wrapper functions for composables with required parameters
- Use meaningful test data in previews

### Documentation
- Add KDoc comments to preview functions - they appear in the browser
- Document props, states, and usage guidelines

```kotlin
/**
 * Primary action button used for main CTAs.
 *
 * Use for: form submissions, primary navigation actions
 * Don't use for: destructive actions (use ErrorButton instead)
 */
@Preview(name = "Primary Button", group = "Buttons")
@Composable
fun PrimaryButtonPreview() {
    PrimaryButton(text = "Submit")
}
```
