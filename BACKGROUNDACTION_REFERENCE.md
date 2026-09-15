# BackgroundAction Tweak Template - Quick Reference

## Location

The complete, buildable Theos project is located at:
```
/home/user/BackgroundAction/
```

This is a **reference implementation** that demonstrates the 3-layer RunningBoard hook architecture described in `RESEARCH_ANALYSIS.md` and the `runningboard-immortal-process` skill.

## What It Does

Prevents specified apps from being backgrounded/suspended when the user presses Home or switches apps. The app stays in foreground state across Respring.

## Project Status

- ✅ Complete Theos project structure with 3 Tweaks
- ✅ Preferences bundle (Settings.app UI) for bundle ID management
- ✅ Full logging infrastructure (/var/mobile/BackgroundAction_debug.log)
- ✅ Cross-process communication via darwin notifications
- ⚠️ **Unverified** - Requires iOS 16.2+ device for selector validation

## Quick Start (Device Required)

### 1. Validate on Target Device

First, confirm RunningBoard selectors match iOS version:

```bash
# On jailbroken iOS 16.2+ device:
ssh root@device.local

# class-dump runningboardd
class-dump /System/Library/PrivateFrameworks/RunningBoard.framework/runningboardd > /tmp/rbd.h

# Check for these selectors:
grep -E "initWithInstance|terminateWithContext|setTaskState" /tmp/rbd.h
```

### 2. Update Selector Names (if different)

Edit these files:
- `RunningBoardHook.xm` - Line 78: `setTaskState:` selector name
- `SpringBoardHook.xm` - Line 24: `deactivate` selector name (likely different)
- `AppHook.xm` - Lines 40/52: UIApplication/UIScene method names

Verify enum value for Running state (currently hardcoded as `2` on line 87).

### 3. Build

```bash
cd /home/user/BackgroundAction

# Requires Theos + arm64/arm64e LLVM toolchain
make package

# Outputs three .deb files in ../Packages/
ls ../Packages/*.deb
```

### 4. Install

```bash
# On device via SSH:
scp ../Packages/*.deb root@device.local:/tmp/
ssh root@device.local

# Install all three Tweaks
dpkg -i /tmp/BackgroundAction*.deb

# Respring automatically runs via after-install hook
```

### 5. Test

```bash
# On device:

# Open Settings.app → BackgroundAction
# Enter bundle IDs (comma-separated): com.google.Drive, com.google.ios.youtube

# Tail debug log:
tail -f /var/mobile/BackgroundAction_debug.log

# Test an app:
# 1. Open app, start upload/playback
# 2. Press Home (backgrounded)
# 3. Check: /var/jb/var/mobile/Library/Logs/ for OS logs
# 4. Respring (killall -9 SpringBoard)
# 5. Verify: Process still running, upload/playback continuing
```

## Architecture

### Layer 1: RunningBoard (daemon-level)
- File: `RunningBoardHook.xm`
- Fixes process taskState = Running
- Blocks terminate requests
- Maintains PID → bundleID mapping

### Layer 2: App Process (app-level)
- File: `AppHook.xm`
- Suppresses UIApplication._applicationDidEnterBackground
- Suppresses UIScene._didEnterBackground
- Only activates for whitelisted bundle IDs

### Layer 3: SpringBoard (UI-level)
- File: `SpringBoardHook.xm`
- Blocks SBApplication.deactivate
- Prevents SpringBoard from issuing descent commands

### Shared Utilities
- File: `Shared.m` + `include/Shared.h`
- App Group NSUserDefaults (cross-process settings)
- Darwin notification dispatch (live setting updates)
- File-based debug logging

### Settings UI
- Directory: `BackgroundActionPrefs/`
- PreferenceLoader bundle for Settings.app
- Simple text input for comma-separated bundle IDs
- No Respring required (darwin notify instantly syncs)

## Known Limitations

1. **Selector Names Vary by iOS Version**
   - Must validate before building
   - Silent failure if wrong (logs to debug file only)

2. **No Protection from Specialized Exploits**
   - Binary patching can bypass all hooks
   - Kernel access allows memory modification
   - Not suitable for security-critical scenarios

3. **Multi-App Interference**
   - Keeping 3+ apps alive simultaneously causes:
     - Sustained high CPU/GPU usage → thermal throttling
     - Battery drain 3-5x normal
     - Audio session conflicts
     - Jetsam scoring distortions (other apps killed unfairly)
   - Recommend: Whitelist only 1 app at a time, auto-timeout after 30 min

4. **Platform Rejection**
   - Paid repos (Cyder/Silo) may reject as "system modification"
   - Not suitable for official release without safeguards

## Improvements Needed (Not Yet Implemented)

- [ ] Simultaneous enable count limit (max 1-2 apps)
- [ ] Auto-timeout after N minutes
- [ ] Toggle switch in Settings for quick disable
- [ ] App list selector instead of text input
- [ ] Thermal/battery monitoring with auto-disable
- [ ] Whitelist persistence validation (is app still installed?)

## Debugging

### Enable Debug Logging
All layers log to: `/var/mobile/BackgroundAction_debug.log`

```bash
# On device:
tail -f /var/mobile/BackgroundAction_debug.log

# Look for:
# - "RunningBoardHook loaded"
# - "tracked handle pid=XXXX bundleID=..."
# - "BLOCKED terminate for..."
# - "suppressed _applicationDidEnterBackground"
# - "BLOCKED deactivate for..."
```

### Confirm Settings Update
```bash
# After Settings change, should see in log:
# "com.mikiyan1978.backgroundaction.prefschanged" notification posted
# "AppHook self bundleID=... whitelisted=1"
```

### View OS-level Logs
```bash
# On device (with unified logging):
log stream --level debug --predicate 'process == "BackgroundActionRB"'
```

## Related Documentation

- `RESEARCH_ANALYSIS.md` - Full technical deep-dive
- `/root/.claude/skills/local/runningboard-immortal-process/` - Implementation skill
- `/root/.claude/skills/local/ios-tweak-paid-release/` - Distribution guide
- `/root/.claude/skills/local/ios-tweak-license-protection/` - Anti-piracy patterns

## Next Steps

1. **On real device**: Validate selectors match iOS version
2. **Fix discrepancies**: Update hook files with actual selector names
3. **Build & test**: Confirm single app scenario works
4. **Iterate**: Test multi-app interference, add safeguards
5. **Publish**: Decide between self-distribution (beta) or platform (release)

---

**Last Updated**: 2025-09-15
**Project Status**: Research Phase Complete → Device Validation Pending
**Requires**: Jailbroken iOS 16.2+ device with Dopamine/roothide
