# ADB Accessibility Debugging Commands

Complete reference for debugging accessibility issues via ADB.

## Quick Reference

| Task | Command |
|------|---------|
| Check TalkBack enabled | `adb shell settings get secure enabled_accessibility_services` |
| Accessibility service info | `adb shell dumpsys accessibility` |
| Input focus state | `adb shell dumpsys input` |
| Window focus | `adb shell dumpsys window` |
| UI hierarchy dump | `adb shell uiautomator dump` |
| Live events | `adb shell uiautomator events` |

## Accessibility Service Diagnostics

### List Enabled Services

```bash
# Get all enabled accessibility services
adb shell settings get secure enabled_accessibility_services

# Example output:
# com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
```

### Detailed Service Info

```bash
# Full accessibility dump
adb shell dumpsys accessibility

# Key sections to look for:
# - "Service[label=TalkBack" - TalkBack configuration
# - "mAccessibilityFocusedWindow" - Current focus
# - "Windows:" - Window accessibility info
```

### TalkBack-Specific Info

```bash
# Filter for TalkBack info
adb shell dumpsys accessibility | grep -A 20 "Service\[label=TalkBack"

# Check TalkBack preferences
adb shell dumpsys accessibility | grep -E "mSpeakElementIds|mVerbosity"
```

## Focus State Analysis

### Accessibility Focus

```bash
# Current accessibility focus
adb shell dumpsys accessibility | grep -A 10 "mAccessibilityFocusedWindow"

# Output shows:
# - Window with a11y focus
# - Node ID that has focus
# - Source of focus event
```

### Input Focus

```bash
# FocusController state
adb shell dumpsys input | grep -A 30 "FocusController"

# Shows:
# - Currently focused window
# - Focus chain
# - Touch mode state
```

### Window Focus

```bash
# Current window focus
adb shell dumpsys window | grep -E "mCurrentFocus|mFocusedApp|mFocusedWindow"

# Activity focus
adb shell dumpsys window | grep -A 5 "mFocusedApp"
```

## Node Tree Inspection

### UI Automator Dump

```bash
# Dump current UI hierarchy to XML
adb shell uiautomator dump /sdcard/ui.xml

# Pull to local machine
adb pull /sdcard/ui.xml

# View inline (requires xmllint)
adb shell uiautomator dump /dev/tty 2>/dev/null | xmllint --format -
```

### Understanding the Dump

```xml
<!-- Example node -->
<node
    index="0"
    text="Submit"
    resource-id="com.example:id/submit_button"
    class="android.widget.Button"
    package="com.example.app"
    content-desc="Submit form"
    checkable="false"
    checked="false"
    clickable="true"
    enabled="true"
    focusable="true"
    focused="false"
    scrollable="false"
    long-clickable="false"
    password="false"
    selected="false"
    bounds="[0,100][200,148]"
/>
```

### Accessibility-Specific Attributes

```bash
# Grep for specific accessibility properties
adb shell uiautomator dump /dev/tty 2>/dev/null | grep -E "content-desc|focusable|clickable"
```

## Live Event Monitoring

### Accessibility Events

```bash
# Watch live accessibility events
adb shell uiautomator events

# Output format:
# TYPE_VIEW_CLICKED [...] contentDescription=Submit
# TYPE_VIEW_ACCESSIBILITY_FOCUSED [...] className=Button
```

### Filtered Event Watching

```bash
# Watch specific event types
adb shell uiautomator events | grep -E "TYPE_VIEW_FOCUSED|TYPE_VIEW_CLICKED"

# Watch events for specific package
adb shell uiautomator events | grep "com.example.app"
```

## Logcat Filters

### Compose Semantics

```bash
# Semantics tree operations
adb logcat -s SemanticsOwner:V

# Compose accessibility bridge
adb logcat | grep -E "ComposeAccessibility|AndroidComposeView"

# Focus traversal
adb logcat | grep -E "FocusTraversal|FocusState|FocusRequester"
```

### TalkBack Processing

```bash
# TalkBack announcements
adb logcat | grep -E "TalkBack|AccessibilityNodeInfo"

# Speech output
adb logcat | grep -E "SpeechController|Utterance"

# Focus feedback
adb logcat | grep "FocusFeedback"
```

### View Accessibility

```bash
# AccessibilityNodeInfo creation
adb logcat | grep "AccessibilityNodeInfo"

# Accessibility events
adb logcat | grep "AccessibilityEvent"

# Window accessibility
adb logcat | grep "AccessibilityWindow"
```

### Enable Verbose Logging

```bash
# Enable verbose semantics logging
adb shell setprop log.tag.SemanticsOwner VERBOSE
adb shell setprop log.tag.AccessibilityNodeInfo VERBOSE
adb shell setprop log.tag.ComposeAccessibility VERBOSE

# Restart app to pick up changes
adb shell am force-stop com.example.app
adb shell monkey -p com.example.app 1
```

## Debugging Scripts

### Complete Accessibility Snapshot

```bash
#!/bin/bash
# save as a11y-snapshot.sh

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_DIR="a11y_debug_$TIMESTAMP"
mkdir -p "$OUTPUT_DIR"

echo "Capturing accessibility snapshot..."

# UI hierarchy
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml "$OUTPUT_DIR/ui_hierarchy.xml"

# Accessibility service state
adb shell dumpsys accessibility > "$OUTPUT_DIR/accessibility_dump.txt"

# Input state
adb shell dumpsys input > "$OUTPUT_DIR/input_dump.txt"

# Window state
adb shell dumpsys window > "$OUTPUT_DIR/window_dump.txt"

# Recent logcat
adb logcat -d | grep -E "Accessibility|Semantics|TalkBack" > "$OUTPUT_DIR/a11y_logcat.txt"

echo "Snapshot saved to $OUTPUT_DIR/"
ls -la "$OUTPUT_DIR/"
```

### Focus Comparison Tool

```bash
#!/bin/bash
# Compare focus state before and after an action

echo "Press Enter to capture BEFORE state..."
read

BEFORE_FOCUS=$(adb shell dumpsys accessibility | grep -A 10 "mAccessibilityFocusedWindow")
BEFORE_INPUT=$(adb shell dumpsys input | grep -A 5 "FocusedWindow")

echo "Perform your action, then press Enter..."
read

AFTER_FOCUS=$(adb shell dumpsys accessibility | grep -A 10 "mAccessibilityFocusedWindow")
AFTER_INPUT=$(adb shell dumpsys input | grep -A 5 "FocusedWindow")

echo "=== A11Y FOCUS CHANGE ==="
echo "BEFORE:"
echo "$BEFORE_FOCUS"
echo ""
echo "AFTER:"
echo "$AFTER_FOCUS"

echo ""
echo "=== INPUT FOCUS CHANGE ==="
echo "BEFORE:"
echo "$BEFORE_INPUT"
echo ""
echo "AFTER:"
echo "$AFTER_INPUT"
```

## Device Settings

### Accessibility Settings via ADB

```bash
# Enable/disable TalkBack
adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
adb shell settings put secure enabled_accessibility_services ""

# Touch exploration (for TalkBack)
adb shell settings put secure touch_exploration_enabled 1
adb shell settings put secure touch_exploration_enabled 0

# Check current settings
adb shell settings list secure | grep accessibility
```

### Developer Options

```bash
# Show layout bounds
adb shell setprop debug.layout true

# Show touch points
adb shell content insert --uri content://settings/system --bind name:s:show_touches --bind value:i:1

# Pointer location
adb shell content insert --uri content://settings/system --bind name:s:pointer_location --bind value:i:1
```

## Troubleshooting Common Issues

### "No accessibility focus"

```bash
# Check if any window has focus
adb shell dumpsys accessibility | grep "mAccessibilityFocusedWindow"

# If null/empty, TalkBack may not be properly enabled
adb shell settings get secure enabled_accessibility_services
```

### "Focus on wrong element"

```bash
# Dump UI and check bounds
adb shell uiautomator dump /dev/tty 2>/dev/null | grep -B 2 -A 2 "focused=\"true\""

# Compare with accessibility focus
adb shell dumpsys accessibility | grep -A 5 "focusedNode"
```

### "Announcements not speaking"

```bash
# Check TalkBack speech settings
adb logcat | grep -E "SpeechController|Utterance|TtsWrapper"

# Verify live region events
adb shell uiautomator events | grep "ANNOUNCEMENT"
```

### "Scroll not working via TalkBack"

```bash
# Check scroll actions on container
adb shell uiautomator dump /dev/tty 2>/dev/null | grep -E "scrollable|ACTION_SCROLL"

# Verify accessibility events
adb shell uiautomator events | grep "TYPE_VIEW_SCROLLED"
```
