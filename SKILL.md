---
name: adb-automation
description: Use when automating Android device interactions via ADB - tapping UI elements, navigating apps, sending messages, opening apps, managing alarms, taking screenshots for visual feedback, sending voice messages, installing apps, or any task that requires controlling an Android phone remotely. Triggers on "android", "adb", "phone", "tap", "swipe", "screenshot", "voice message", "telegram", "alarmy", "install app", "open app", "send message", "navigate", "UI automation".
---

# ADB Automation Skill

Comprehensive skill for controlling an Android device via ADB. Covers UI navigation, app interaction, messaging, voice messages, file management, and device configuration.

## Prerequisites

- **adb** — Android Debug Bridge. Install via [Android platform-tools](https://developer.android.com/tools/releases/platform-tools) or `brew install android-platform-tools`
- **python3** — For parsing UI hierarchy XML dumps
- **edge-tts** — For natural TTS voice messages. Install: `pip3 install edge-tts`
- **ffmpeg** — For audio format conversion. Install: `brew install ffmpeg` or `apt install ffmpeg`

The last two are only required for the voice message workflow. All other automation works with just `adb` and `python3`.

### Android Device Setup

1. Enable **Developer Options** on the device (tap Build Number 7 times in Settings > About)
2. Enable **USB Debugging** in Developer Options
3. Connect via USB and authorize the computer when prompted
4. Verify: `adb devices -l` should show your device as `device`

## Core Principles

### Always Start With Device Setup

Before any automation session, optimize the device for reliable ADB control:

```bash
DEVICE="<your-device-serial>"  # Find with: adb devices -l

# Lock portrait orientation
adb -s $DEVICE shell settings put system accelerometer_rotation 0
adb -s $DEVICE shell settings put system user_rotation 0

# Disable animations (critical for reliable UI automation)
adb -s $DEVICE shell settings put global window_animation_scale 0
adb -s $DEVICE shell settings put global transition_animation_scale 0
adb -s $DEVICE shell settings put global animator_duration_scale 0

# Prevent screen timeout (30 min)
adb -s $DEVICE shell settings put system screen_off_timeout 1800000

# Fixed brightness (no auto-adjust)
adb -s $DEVICE shell settings put system screen_brightness_mode 0

# Enable Do Not Disturb (no interruptions during automation)
adb -s $DEVICE shell settings put global zen_mode 1
```

### Session Teardown (Restore After Automation)

```bash
# Restore animations
adb -s $DEVICE shell settings put global window_animation_scale 1
adb -s $DEVICE shell settings put global transition_animation_scale 1
adb -s $DEVICE shell settings put global animator_duration_scale 1

# Restore auto-rotation
adb -s $DEVICE shell settings put system accelerometer_rotation 1

# Restore auto-brightness
adb -s $DEVICE shell settings put system screen_brightness_mode 1

# Disable DND
adb -s $DEVICE shell settings put global zen_mode 0

# Restore screen timeout (default 30s)
adb -s $DEVICE shell settings put system screen_off_timeout 30000
```

### Coordinate System

The device screen resolution and screenshot resolution differ. **Always verify resolution first:**

```bash
adb -s $DEVICE shell wm size
# Example output: Physical size: 1200x2670
```

ADB input commands use **device coordinates** (e.g., 1200x2670). Screenshots rendered in the conversation may appear at a different pixel size. If you calculate coordinates from a screenshot image, scale accordingly:

```
device_x = (screenshot_x / screenshot_width) * device_width
device_y = (screenshot_y / screenshot_height) * device_height
```

**Best practice:** Use `uiautomator dump` to get exact element bounds whenever possible — this gives device coordinates directly, no scaling needed.

---

## UI Inspection

### Screenshot (Visual Feedback)

Take a screenshot and read it to see the current screen state:

```bash
adb -s $DEVICE shell screencap -p /sdcard/screen.png
adb -s $DEVICE pull /sdcard/screen.png /tmp/screen.png
```

Then use the Read tool on `/tmp/screen.png` to visually inspect the screen.

### UI Automator Dump (Element Bounds)

For precise tapping, dump the UI hierarchy to get exact element coordinates:

```bash
adb -s $DEVICE shell uiautomator dump /sdcard/ui.xml
adb -s $DEVICE pull /sdcard/ui.xml /tmp/ui.xml
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('/tmp/ui.xml')
for elem in tree.iter('node'):
    text = elem.get('text', '')
    desc = elem.get('content-desc', '')
    bounds = elem.get('bounds', '')
    clickable = elem.get('clickable', '')
    cls = elem.get('class', '')
    if text or desc:
        print(f'text={text!r} desc={desc!r} click={clickable} bounds={bounds}')
"
```

**Important limitations:**
- Apps using custom rendering (Telegram chat list, games) won't expose elements in uiautomator. Fall back to screenshots + coordinate calculation.
- WebView content is often invisible to uiautomator.
- Always combine both methods: screenshot for visual context, uiautomator for precise bounds.

### Finding Focused App

```bash
adb -s $DEVICE shell dumpsys window | grep "mCurrentFocus\|mFocusedApp"
```

### Check Lock Screen State

```bash
adb -s $DEVICE shell dumpsys window | grep "mDreamingLockscreen\|mShowingLockscreen\|showing="
```

---

## UI Interaction

### Tapping

```bash
# Tap at exact coordinates (from uiautomator bounds)
adb -s $DEVICE shell input tap 600 1500

# When you have bounds like [252,1948][1140,2013], tap the center:
# x = (252 + 1140) / 2 = 696, y = (1948 + 2013) / 2 = 1980
adb -s $DEVICE shell input tap 696 1980
```

### Swiping / Scrolling

```bash
# Swipe up (scroll down): from bottom to top
adb -s $DEVICE shell input swipe 600 2000 600 800 300

# Swipe down (scroll up): from top to bottom
adb -s $DEVICE shell input swipe 600 800 600 2000 300

# Gentle scroll (shorter distance)
adb -s $DEVICE shell input swipe 400 1800 400 1200 200
```

### Long Press

```bash
# Long press = swipe with zero distance and long duration
adb -s $DEVICE shell input swipe 600 1500 600 1500 1500
```

### Text Input

```bash
# ASCII text only (no spaces directly)
adb -s $DEVICE shell input text "hello"

# Spaces use %s
adb -s $DEVICE shell input text "hello%sworld"

# LIMITATION: Cyrillic/Unicode does NOT work with input text
# Use intent-based approaches for non-ASCII (see Voice Messages section)
```

### Key Events

```bash
adb -s $DEVICE shell input keyevent KEYCODE_BACK        # Back
adb -s $DEVICE shell input keyevent KEYCODE_HOME        # Home
adb -s $DEVICE shell input keyevent KEYCODE_ENTER       # Enter
adb -s $DEVICE shell input keyevent KEYCODE_DEL         # Backspace
adb -s $DEVICE shell input keyevent 82                  # Menu / Unlock
```

### Unlock Screen

```bash
# Wake screen
adb -s $DEVICE shell input keyevent KEYCODE_WAKEUP
# Swipe up to dismiss lock screen (if no PIN)
adb -s $DEVICE shell input swipe 540 2000 540 800 300
# Or use menu key
adb -s $DEVICE shell input keyevent 82
```

---

## App Management

### Launch Apps

```bash
# By package name (most reliable)
adb -s $DEVICE shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1

# By activity name
adb -s $DEVICE shell am start -n com.example.app/.MainActivity
```

### Find Package Names

```bash
# Search installed packages
adb -s $DEVICE shell pm list packages | grep -i telegram
# Example: package:org.telegram.messenger

# List third-party apps only
adb -s $DEVICE shell pm list packages -3
```

### Install from Play Store

```bash
# Open app's Play Store page
adb -s $DEVICE shell am start -a android.intent.action.VIEW \
    -d "market://details?id=org.telegram.messenger"
```

### Common App Packages

| App | Package |
|-----|---------|
| Telegram | `org.telegram.messenger` |
| Claude | `com.anthropic.claude` |
| Google Keep | `com.google.android.keep` |
| Alarmy | `droom.sleepIfUCan` |
| Chrome | `com.android.chrome` |
| Settings | `com.android.settings` |
| Play Store | `com.android.vending` |

---

## Telegram Automation

### Open a Chat via Search

1. Launch Telegram:
   ```bash
   adb -s $DEVICE shell monkey -p org.telegram.messenger -c android.intent.category.LAUNCHER 1
   ```

2. Tap search icon (find bounds with uiautomator, typically `desc='Search'`):
   ```bash
   adb -s $DEVICE shell input tap <search_x> <search_y>
   ```

3. Type contact name (ASCII only):
   ```bash
   adb -s $DEVICE shell input text "ContactName"
   ```

4. Hide keyboard, then tap the contact:
   ```bash
   adb -s $DEVICE shell input keyevent KEYCODE_BACK   # hide keyboard
   adb -s $DEVICE shell input tap <contact_x> <contact_y>
   ```

### Send a Text Message

```bash
# In an open chat, tap message field (find via uiautomator: text='Message')
adb -s $DEVICE shell input tap <msg_field_x> <msg_field_y>
adb -s $DEVICE shell input text "Hello%sfrom%sADB"
# Tap send button
adb -s $DEVICE shell input tap <send_x> <send_y>
```

### Send Non-ASCII Text (Ukrainian, etc.)

Use Android's share intent — it handles Unicode properly:

```bash
adb -s $DEVICE shell "am start -a android.intent.action.SEND \
    -t text/plain \
    --es android.intent.extra.TEXT 'Привіт! Як справи?' \
    -p org.telegram.messenger"
```

This opens a chat picker. Search for the contact, select them, and tap send.

### Delete Messages

1. Long-press a message to enter selection mode:
   ```bash
   adb -s $DEVICE shell input swipe <x> <y> <x> <y> 1500
   ```

2. Tap additional messages to select them

3. Tap the trash/delete icon (find via uiautomator)

4. In the confirmation dialog, optionally check "Also delete for..." and tap Delete

### Send Files from Chat

1. Open the target chat
2. Tap attach button (uiautomator: `desc='Attach media'`)
3. Tap **File** tab (uiautomator: `text='File'`)
4. File appears under "Recent files" if placed in `/sdcard/Download/`
5. Tap the file to select it
6. Tap "Send 1 file" button

---

## Voice Messages via TTS

Generate natural-sounding voice messages using `edge-tts` and send via Telegram.

### Setup (One-Time)

```bash
pip3 install edge-tts
```

### Generate and Send Voice Message

```bash
# 1. Generate with natural male voice
edge-tts --voice "en-US-AndrewNeural" \
    --text "Hey, how are you doing? Hope everything is going well!" \
    --write-media /tmp/voice.mp3

# 2. Convert to Telegram voice format (ogg opus)
ffmpeg -y -i /tmp/voice.mp3 -c:a libopus -b:a 64k -ar 48000 -ac 1 /tmp/voice.ogg

# 3. Push to phone
adb -s $DEVICE push /tmp/voice.ogg /sdcard/Download/voice.ogg

# 4. Open target chat in Telegram, then:
#    Attach > File tab > tap voice.ogg under "Recent files" > Send
```

### Recommended Voices

| Voice | Language | Style |
|-------|----------|-------|
| `en-US-AndrewNeural` | English | Warm, confident, authentic |
| `en-US-BrianNeural` | English | Approachable, casual |
| `en-GB-RyanNeural` | English | British male |
| `uk-UA-OstapNeural` | Ukrainian | Male, friendly |
| `uk-UA-PolinaNeural` | Ukrainian | Female, friendly |
| `de-DE-ConradNeural` | German | Male |
| `fr-FR-HenriNeural` | French | Male |
| `ja-JP-KeitaNeural` | Japanese | Male |

List all voices: `edge-tts --list-voices`

### Alternative: macOS `say` (Faster, Lower Quality)

```bash
say -v Daniel "Hello there" -o /tmp/msg.aiff
ffmpeg -y -i /tmp/msg.aiff -c:a libopus -b:a 32k -ar 48000 -ac 1 /tmp/msg.ogg
```

---

## Google Keep Notes

### Create a Note

```bash
adb -s $DEVICE shell "am start -a android.intent.action.SEND \
    -t text/plain \
    --es android.intent.extra.TEXT 'Note content here' \
    --es android.intent.extra.SUBJECT 'Note title' \
    -p com.google.android.keep"
```

Then tap "Save" in the popup.

### Delete a Note

Open Google Keep, find the note visually (screenshot), tap it, then use the menu (three dots) to delete.

---

## Alarmy (Alarm Management)

Package: `droom.sleepIfUCan`

### Open Alarmy

```bash
adb -s $DEVICE shell monkey -p droom.sleepIfUCan -c android.intent.category.LAUNCHER 1
```

### Navigate to Alarm Tab

Tap the "Alarm" tab at the bottom (find via uiautomator or screenshot).

### Edit an Alarm (Time Picker)

Alarmy uses a scroll-wheel time picker. The three columns are:

- **Hours** (left): center x from uiautomator bounds
- **Minutes** (center): center x from uiautomator bounds
- **AM/PM** (right): center x from uiautomator bounds

Each column row is ~174px tall. To change values:

```bash
# Scroll down (decrease value): swipe from above center to below
adb -s $DEVICE shell input swipe <col_x> 950 <col_x> 1200 300

# Scroll up (increase value): swipe from below center to above
adb -s $DEVICE shell input swipe <col_x> 1200 <col_x> 700 200
```

Each swipe moves approximately 1 step. Repeat as needed.

### Toggle Daily / One-Time

Find the "Daily" checkbox via uiautomator and tap it to toggle between one-time and daily repeat.

### Save Alarm

Find the "Save" button via uiautomator (typically at bottom center) and tap.

---

## File Transfer

### Push Files to Device

```bash
adb -s $DEVICE push /local/path/file.ogg /sdcard/Download/file.ogg
```

### Pull Files from Device

```bash
adb -s $DEVICE pull /sdcard/some_file.txt /local/path/
```

### Make Files Visible to Apps

Place files in `/sdcard/Download/` for best visibility in file pickers. Trigger media scan if needed:

```bash
adb -s $DEVICE shell am broadcast \
    -a android.intent.action.MEDIA_SCANNER_SCAN_FILE \
    -d file:///sdcard/Download/file.ogg
```

---

## Permissions Management

### Grant Permissions via ADB

```bash
adb -s $DEVICE shell pm grant <package> android.permission.CAMERA
adb -s $DEVICE shell pm grant <package> android.permission.READ_EXTERNAL_STORAGE
adb -s $DEVICE shell pm grant <package> android.permission.RECORD_AUDIO
```

### Handle Permission Dialogs

When a permission dialog appears, use uiautomator to find buttons like "While using the app", "Allow", "Don't allow" and tap them.

---

## Workflow Patterns

### Reliable App Navigation Pattern

```
1. Launch app via monkey command
2. Wait 2-3 seconds for app to load
3. Take screenshot to verify state
4. Use uiautomator dump for element bounds
5. Tap target element using exact bounds
6. Take screenshot to verify result
7. Repeat from step 4 for next action
```

### Handling Dialogs and Popups

Apps frequently show permission dialogs, update prompts, onboarding overlays, and ads. When automation gets stuck:

1. Take a screenshot to see what's blocking
2. Look for dismiss buttons ("Not now", "Cancel", "Skip", "Got it", "OK")
3. Use uiautomator to find and tap dismiss buttons
4. If it's a system permission dialog, grant or deny as appropriate
5. Resume the original workflow

### Recovery from Wrong Screen

If you end up on the wrong screen:

```bash
# Go back
adb -s $DEVICE shell input keyevent KEYCODE_BACK

# Go home
adb -s $DEVICE shell input keyevent KEYCODE_HOME

# Force stop and relaunch
adb -s $DEVICE shell am force-stop <package>
adb -s $DEVICE shell monkey -p <package> -c android.intent.category.LAUNCHER 1
```

---

## Troubleshooting

### Device Not Responding to Taps

- Verify coordinates are in device resolution (not screenshot resolution)
- Check for overlay dialogs blocking input
- Ensure animations are disabled
- Try `adb -s $DEVICE shell input keyevent KEYCODE_WAKEUP` if screen is off

### Can't Type Non-ASCII Text

- `adb shell input text` only supports ASCII
- Use share intents for Unicode text
- For clipboard-based approaches, consider ADBKeyboard or similar input methods

### uiautomator Dump Fails or Returns Empty

- Some apps use custom rendering (games, Flutter, React Native)
- Fall back to screenshot + coordinate calculation
- Try `adb shell uiautomator dump --compressed /sdcard/ui.xml` for faster dumps

### App Opens but Shows Wrong Screen

- The app may have been in a different state. Use `am force-stop` first
- Check if the app requires login or onboarding completion
- Navigate back to the expected screen manually

For detailed ADB reference, see `references/commands.md`.
