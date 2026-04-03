# ADB Automation Quick Reference

## Device Setup Commands

```bash
# Verify connection
adb devices -l

# Get device info
adb -s $DEVICE shell getprop ro.product.model
adb -s $DEVICE shell getprop ro.build.version.release
adb -s $DEVICE shell wm size           # Screen resolution
adb -s $DEVICE shell wm density        # DPI
```

## Automation-Optimized Settings

### Enable (Before Automation)

```bash
# Portrait lock
adb -s $DEVICE shell settings put system accelerometer_rotation 0
adb -s $DEVICE shell settings put system user_rotation 0

# Kill animations
adb -s $DEVICE shell settings put global window_animation_scale 0
adb -s $DEVICE shell settings put global transition_animation_scale 0
adb -s $DEVICE shell settings put global animator_duration_scale 0

# Long screen timeout
adb -s $DEVICE shell settings put system screen_off_timeout 1800000

# Fixed brightness
adb -s $DEVICE shell settings put system screen_brightness_mode 0
adb -s $DEVICE shell settings put system screen_brightness 200

# Do Not Disturb
adb -s $DEVICE shell settings put global zen_mode 1

# Show pointer location (debug coordinate issues)
adb -s $DEVICE shell settings put system pointer_location 1

# Show touches (visual feedback)
adb -s $DEVICE shell settings put system show_touches 1

# Stay awake while charging
adb -s $DEVICE shell settings put global stay_on_while_plugged_in 3
```

### Restore (After Automation)

```bash
adb -s $DEVICE shell settings put global window_animation_scale 1
adb -s $DEVICE shell settings put global transition_animation_scale 1
adb -s $DEVICE shell settings put global animator_duration_scale 1
adb -s $DEVICE shell settings put system accelerometer_rotation 1
adb -s $DEVICE shell settings put system screen_brightness_mode 1
adb -s $DEVICE shell settings put global zen_mode 0
adb -s $DEVICE shell settings put system screen_off_timeout 30000
adb -s $DEVICE shell settings put system pointer_location 0
adb -s $DEVICE shell settings put system show_touches 0
adb -s $DEVICE shell settings put global stay_on_while_plugged_in 0
```

## Screen Inspection

```bash
# Screenshot
adb -s $DEVICE shell screencap -p /sdcard/screen.png
adb -s $DEVICE pull /sdcard/screen.png /tmp/screen.png

# UI hierarchy dump
adb -s $DEVICE shell uiautomator dump /sdcard/ui.xml
adb -s $DEVICE pull /sdcard/ui.xml /tmp/ui.xml

# Current foreground app
adb -s $DEVICE shell dumpsys window | grep "mCurrentFocus"

# Screen on/off state
adb -s $DEVICE shell dumpsys power | grep "mWakefulness"
```

## UI Hierarchy Parser (Python)

```python
import xml.etree.ElementTree as ET

tree = ET.parse('/tmp/ui.xml')
for elem in tree.iter('node'):
    text = elem.get('text', '')
    desc = elem.get('content-desc', '')
    bounds = elem.get('bounds', '')
    clickable = elem.get('clickable', '')
    cls = elem.get('class', '')
    checked = elem.get('checked', '')
    scrollable = elem.get('scrollable', '')
    if text or desc:
        print(f'text={text!r} desc={desc!r} click={clickable} bounds={bounds}')
```

### Parse bounds to get center coordinates:

```python
import re

def bounds_center(bounds_str):
    """Parse '[x1,y1][x2,y2]' and return center (cx, cy)."""
    nums = list(map(int, re.findall(r'\d+', bounds_str)))
    return (nums[0] + nums[2]) // 2, (nums[1] + nums[3]) // 2
```

## Input Commands

```bash
# Tap
adb -s $DEVICE shell input tap <x> <y>

# Swipe (scroll)
adb -s $DEVICE shell input swipe <x1> <y1> <x2> <y2> <duration_ms>

# Long press
adb -s $DEVICE shell input swipe <x> <y> <x> <y> 1500

# Type ASCII text (%s = space)
adb -s $DEVICE shell input text "hello%sworld"

# Key events
adb -s $DEVICE shell input keyevent KEYCODE_BACK          # 4
adb -s $DEVICE shell input keyevent KEYCODE_HOME          # 3
adb -s $DEVICE shell input keyevent KEYCODE_ENTER         # 66
adb -s $DEVICE shell input keyevent KEYCODE_DEL           # 67
adb -s $DEVICE shell input keyevent KEYCODE_WAKEUP        # 224
adb -s $DEVICE shell input keyevent KEYCODE_APP_SWITCH    # 187
adb -s $DEVICE shell input keyevent KEYCODE_MENU          # 82
adb -s $DEVICE shell input keyevent KEYCODE_MOVE_HOME     # 122
adb -s $DEVICE shell input keyevent KEYCODE_MOVE_END      # 123

# Select all text (Ctrl+A equivalent)
adb -s $DEVICE shell input keyevent KEYCODE_MOVE_HOME
adb -s $DEVICE shell input keyevent --longpress KEYCODE_SHIFT_LEFT KEYCODE_MOVE_END
adb -s $DEVICE shell input keyevent KEYCODE_DEL
```

## App Launchers

```bash
# Generic launcher (most reliable)
adb -s $DEVICE shell monkey -p <package> -c android.intent.category.LAUNCHER 1

# Specific activity
adb -s $DEVICE shell am start -n <package>/<activity>

# With intent data
adb -s $DEVICE shell am start -a android.intent.action.VIEW -d "https://example.com"

# Open Play Store page
adb -s $DEVICE shell am start -a android.intent.action.VIEW \
    -d "market://details?id=<package>"

# Share text to app
adb -s $DEVICE shell "am start -a android.intent.action.SEND \
    -t text/plain \
    --es android.intent.extra.TEXT 'message text' \
    -p <package>"

# Share file to app
adb -s $DEVICE shell "am start -a android.intent.action.SEND \
    -t audio/ogg \
    --eu android.intent.extra.STREAM file:///sdcard/file.ogg \
    -p <package>"

# Force stop
adb -s $DEVICE shell am force-stop <package>

# Clear app data
adb -s $DEVICE shell pm clear <package>
```

## File Operations

```bash
# Push to device
adb -s $DEVICE push /local/file /sdcard/Download/file

# Pull from device
adb -s $DEVICE pull /sdcard/file /local/path/

# Copy on device
adb -s $DEVICE shell cp /sdcard/source /sdcard/dest

# List files
adb -s $DEVICE shell ls -la /sdcard/Download/

# Trigger media scanner
adb -s $DEVICE shell am broadcast \
    -a android.intent.action.MEDIA_SCANNER_SCAN_FILE \
    -d file:///sdcard/Download/file.ogg
```

## Voice Message Generation

```bash
# English (natural male)
edge-tts --voice "en-US-AndrewNeural" \
    --text "Your message here" \
    --write-media /tmp/voice.mp3

# Ukrainian (male)
edge-tts --voice "uk-UA-OstapNeural" \
    --text "Ваше повідомлення тут" \
    --write-media /tmp/voice.mp3

# Convert to Telegram voice format
ffmpeg -y -i /tmp/voice.mp3 -c:a libopus -b:a 64k -ar 48000 -ac 1 /tmp/voice.ogg

# Push and make available
adb -s $DEVICE push /tmp/voice.ogg /sdcard/Download/voice.ogg

# List all available voices
edge-tts --list-voices

# List voices for a specific language
edge-tts --list-voices | grep "uk-UA"
```

## Package Discovery

```bash
# All packages
adb -s $DEVICE shell pm list packages

# Third-party only
adb -s $DEVICE shell pm list packages -3

# Search by name
adb -s $DEVICE shell pm list packages | grep -i <name>

# Get APK path
adb -s $DEVICE shell pm path <package>

# Grant permission
adb -s $DEVICE shell pm grant <package> android.permission.<PERM>
```

## Battery & Power

```bash
adb -s $DEVICE shell dumpsys battery
adb -s $DEVICE shell dumpsys battery | grep -E "level|status|health|temperature"
```

## Network

```bash
# WiFi state
adb -s $DEVICE shell dumpsys wifi | grep "Wi-Fi is"

# IP address
adb -s $DEVICE shell ip addr show wlan0

# Toggle airplane mode
adb -s $DEVICE shell settings put global airplane_mode_on 1
adb -s $DEVICE shell am broadcast -a android.intent.action.AIRPLANE_MODE
```

## Notifications

```bash
# Expand notification shade
adb -s $DEVICE shell cmd statusbar expand-notifications

# Collapse
adb -s $DEVICE shell cmd statusbar collapse
```
