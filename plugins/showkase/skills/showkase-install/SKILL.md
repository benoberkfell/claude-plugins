---
name: showkase-install
description: Install and configure Showkase in an Android project. Use when adding Showkase library, setting up dependencies, configuring build.gradle, or integrating Showkase into a new or existing Android/Compose project.
allowed-tools: Read, Glob, Grep, Edit, Write
---

# Showkase Installation

Install and configure Showkase, Airbnb's annotation-processor library for organizing and visualizing Jetpack Compose UI elements.

## Prerequisites

Before installing Showkase, ensure the project:
- Uses Jetpack Compose
- Has Kotlin configured
- Uses either KSP or KAPT annotation processing

## Installation Steps

### Step 1: Add Dependencies

Add these dependencies to the module's `build.gradle` or `build.gradle.kts` file.

**For debug-only builds (recommended):**

```kotlin
// build.gradle.kts
dependencies {
    debugImplementation("com.airbnb.android:showkase:1.0.5")
    implementation("com.airbnb.android:showkase-annotation:1.0.5")
    kspDebug("com.airbnb.android:showkase-processor:1.0.5")
}
```

Or with KAPT:
```kotlin
dependencies {
    debugImplementation("com.airbnb.android:showkase:1.0.5")
    implementation("com.airbnb.android:showkase-annotation:1.0.5")
    kaptDebug("com.airbnb.android:showkase-processor:1.0.5")
}
```

**For release builds as well:**
```kotlin
dependencies {
    implementation("com.airbnb.android:showkase:1.0.5")
    ksp("com.airbnb.android:showkase-processor:1.0.5")
}
```

### Step 2: Create Root Module

Create a class implementing `ShowkaseRootModule` with the `@ShowkaseRoot` annotation in the **root module** of the app:

```kotlin
import com.airbnb.android.showkase.annotation.ShowkaseRoot
import com.airbnb.android.showkase.annotation.ShowkaseRootModule

@ShowkaseRoot
class MyAppShowkaseRoot : ShowkaseRootModule
```

Place this in a logical location like:
- `app/src/debug/java/com/yourpackage/ShowkaseRoot.kt` (debug only)
- `app/src/main/java/com/yourpackage/ShowkaseRoot.kt` (all builds)

### Step 3: Add Browser Launch Point

Add a way to launch the Showkase browser, typically in a debug menu or settings screen:

```kotlin
import com.airbnb.android.showkase.models.Showkase

// Launch the browser
startActivity(Showkase.getBrowserIntent(context))
```

### Step 4: Build the Project

Build the project to generate the Showkase code. The `getBrowserIntent` extension function is generated during compilation.

## Multi-Module Setup

For multi-module projects:

1. Add `showkase-annotation` dependency to **all modules** with UI elements
2. Add `showkase-processor` to **all modules** with annotated elements
3. Add the `@ShowkaseRoot` class only in the **root/app module**

```kotlin
// In feature modules
dependencies {
    implementation("com.airbnb.android:showkase-annotation:1.0.5")
    kspDebug("com.airbnb.android:showkase-processor:1.0.5")
}

// In app module (additionally)
dependencies {
    debugImplementation("com.airbnb.android:showkase:1.0.5")
}
```

## Optional Configuration

### Skip Private Previews

To skip private `@Preview` functions during compilation:

**KSP:**
```kotlin
ksp {
    arg("skipPrivatePreviews", "true")
}
```

**KAPT:**
```kotlin
kapt {
    arguments {
        arg("skipPrivatePreviews", "true")
    }
}
```

### Custom Preview Annotations (KAPT only)

For custom multi-preview annotations with KAPT:

```kotlin
kapt {
    arguments {
        arg("multiPreviewType", "com.yourpackage.YourCustomPreview")
    }
}
```

## Verification

After installation, verify by:
1. Building the project successfully
2. Checking that `Showkase.getBrowserIntent()` is available
3. Launching the browser activity

## Troubleshooting

- **getBrowserIntent not found**: Rebuild the project; it's generated during annotation processing
- **No components showing**: Ensure composables have `@Preview` or `@ShowkaseComposable` annotations
- **Multi-module issues**: Verify the root module has dependencies on all feature modules
