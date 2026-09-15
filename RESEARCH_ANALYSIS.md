# iOS Jailbreak Tweak Development: Process Immortalization & Code Injection Analysis

## Executive Summary

This document summarizes comprehensive research into advanced iOS jailbreak Tweak development, specifically:

1. **RunningBoard Process State Management** - How to keep apps alive across Respring/reboot
2. **Dylib Injection Mechanisms** - Non-jailbreak code injection techniques (TrollFools analysis)
3. **BackgroundAction Reference Implementation** - A reusable Theos template for forcing apps into foreground state

**Key Finding**: Complete process protection is impossible in jailbreak contexts (root privileges + kernel access allow any bypass). The goal is to raise the cost of piracy while protecting legitimate revenue through layered defense.

---

## Part 1: RunningBoard Architecture & Process Immortalization

### Background: The Respring Problem

Modern iOS manages process lifecycle through **RunningBoard daemon** (`runningboardd`):
- Monitors process states: Suspended, Background, Foreground
- Manages resource allocation (CPU, memory, I/O)
- Handles jetsam decisions under memory pressure
- Controls audio background modes, network access, location services

When a user presses Home or closes an app:
1. SpringBoard sends deactivation request
2. RunningBoard updates process taskState
3. FrontBoard/BackBoard trigger UIKit lifecycle callbacks (_didEnterBackground)
4. Process enters Suspended state → CPU clock-gated → minimal battery drain

**The Challenge**: Apps performing critical tasks (music playback, uploads, location tracking) stop working immediately.

### RoadRunner Case Study: Audio Playback Survival

RoadRunner is a commercial Tweak that keeps music playing across Respring. Reverse-engineering shows:

**Architecture**:
- **Layer 1** (RunningBoard): MSHookMessageEx on RBSProcessHandle/RBProcess/RBSProcessState
  - Fixes `taskState` to `2` (Running equivalent) before state queries
  - Intercepts `terminateWithContext:` to prevent jetsam kill
  - Uses **Associated Objects** (objc_setAssociatedObject) to store "immortal" flag per instance
  
- **Layer 2** (App Process): UIApplication/UIScene lifecycle suppression
  - Suppresses `_applicationDidEnterBackground` callback
  - Prevents app from voluntarily stopping audio session
  
- **Layer 3** (SpringBoard): SBApplication/SBMediaController interception
  - Restores Now Playing UI after Respring
  - Re-establishes media control routes via MediaRemote framework

**Critical Detail: Associated Objects are Instance-Level**

The "immortal" flag is NOT persisted in RunningBoard's database. Instead:
- Set as instance field via `objc_setAssociatedObject`
- Lives only in `runningboardd` process memory
- Dies when `runningboardd` exits (rare)
- Post-Respring recovery via **boomerang mechanism**:
  - XPC communication with launchd
  - Mach port inheritance survives userspace reboot
  - RunningBoard re-attaches to previously immortal processes
  - RRManager.reattachImmortalProcess() re-marks them immortal

**Why This Works**: The userspace-reboot (RB2_USERREBOOT) doesn't kill `runningboardd` itself; it's part of kernel-privileged services. The Mach port-based IPC survives, allowing recovery.

### Process State Machine

```
Suspended (3)
    ↓
Background (1)
    ↓
Foreground (0)
    ↓
Running (2) ← Immortal processes stay here
```

Key RunningBoard classes:
- `RBSProcessHandle`: Process identity + bundleID + PID mapping
- `RBProcess`: High-level process lifecycle manager
  - `-terminateWithContext:` - Kill decision point
  - `-processIdentifier` - Returns PID
  
- `RBSProcessState`: Per-process state snapshot
  - `-setTaskState:` - Transition target (0-3)
  - `-bundleID` - Bundle identifier

### Limitations of Process-State Fixes

**What works**:
- Audio playback (already has background mode entitlement)
- Location tracking (with NSLocationAlwaysAndWhenInUseUsageDescription)
- Generic background task assertions (NSBackgroundTaskAssertion)

**What doesn't work**:
- Video playback voluntarily pauses (not RunningBoard's decision - app pauses on state transition)
- Uploads interrupted (NSURLSession backgroundSessionConfiguration has built-in limits)
- Graphics rendering (GPU clock-gated, can't force full power in background)

**Why**: RunningBoard only controls scheduling priority + resource allocation. App-level decisions (like video pause on backgrounding) happen in UIKit via state callbacks, which must be intercepted at Layer 2 for full effect.

---

## Part 2: TrollFools - Non-Jailbreak Code Injection

TrollFools is a sophisticated tool for injecting dylibs into App Store apps without jailbreak. Unlike jailbreak injection (which has system-wide access), TrollFools works under strict Apple sandboxing.

### Mach-O Binary Manipulation

**Architecture**:
```
InjectorV3.swift (Core)
  ├─ locateExecutableInBundle() - Find main binary
  ├─ locateFrameworksDirectoryInBundle() - Assets location
  ├─ identifierOfBundle() / teamIdentifierOfMachO()
  └─ Logging + temporary directory management

InjectorV3+Inject.swift (Pipeline)
  ├─ preprocessAssets() - Filter dylibs/frameworks
  ├─ detectProtectedMachO() - Check code signing (CS_RESTRICT, CS_HARD, CS_KILL flags)
  ├─ cmdInsertLoadCommandRuntimePath() - Add @executable_path/Frameworks
  ├─ cmdInsertLoadCommandDylib() - Insert LC_LOAD_DYLIB / LC_LOAD_WEAK_DYLIB
  └─ Atomic error handling (makeAlternate/restoreAlternate)
```

**Key Techniques**:

1. **Mach-O Load Command Insertion**
   - Dylib binaries are not directly modified
   - Instead, the main app binary's load commands are rewritten
   - New `LC_LOAD_DYLIB` header added pointing to injected dylib
   - Dylib file placed at `@executable_path/Frameworks/`

2. **Weak Reference Option**
   - `LC_LOAD_WEAK_DYLIB` instead of `LC_LOAD_DYLIB`
   - If dylib not found at runtime, app doesn't crash
   - Allows graceful degradation

3. **Automatic Substrate Embedding**
   - TrollFools automatically includes Cydia Substrate.framework
   - This allows Theos Logos hooks to work in non-jailbreak context
   - Substrate.framework performs runtime method swizzling via MSHookMessageEx

4. **Protection Detection**
   - Checks binary for hardened runtime (CS_HARD flag)
   - Detects RESTRICT entitlements
   - Auto-selects between multiple binary slices (arm64/arm64e)
   - For arm64e (A15+, PAC enabled), requires additional considerations

5. **Error Recovery**
   - Atomic operations: create alternate, test, commit or rollback
   - All-or-nothing: if injection fails mid-process, original binary restored
   - No partial corrupted state

### Non-Jailbreak Constraints

Unlike jailbreak (which has root), TrollFools must work within:
- **Sandbox isolation**: Can't directly modify system binaries
- **Code signing**: Apple won't resign modified app
- **Entitlements**: Can only add entitlements already present in original
- **Platform restrictions**: Can't use private frameworks not linked in original app

### Why TrollFools Works

1. The modified app binary is **sideloaded** via TrollStore (which has special entitlements)
2. No system-wide code injection - just this one app gets the dylib
3. Dylib runs within the same sandbox as the app
4. No RunningBoard access (non-jailbreak iOS lacks root)
5. App must be re-signed with a valid developer certificate (free personal signing works)

---

## Part 3: BackgroundAction - Reference Implementation

### Concept

A **Theos Tweak template** that forces any jailbroken app to stay in foreground state:

**Input**: User specifies bundle ID(s) via Settings app
**Output**: Those apps never background - Respring/Home button don't suspend them

### Architecture (3-Layer Defense)

```
Layer 1: RunningBoard Hook (RunningBoardHook.xm)
  - Target: com.apple.runningboardd
  - Action: Force taskState=2 (Running), block terminateWithContext:
  
Layer 2: App-Level Hook (AppHook.xm)
  - Target: All processes (filter wildcard)
  - Action: Suppress UIApplication._applicationDidEnterBackground
  
Layer 3: SpringBoard Hook (SpringBoardHook.xm)
  - Target: com.apple.springboard
  - Action: Block SBApplication.deactivate
```

### Implementation Files

**Core Logic**:
- `RunningBoardHook.xm` (96 lines) - Persistent PID→bundleID mapping + state fixing
- `AppHook.xm` (61 lines) - UIKit lifecycle suppression
- `SpringBoardHook.xm` (40 lines) - Deactivation blocking

**Shared Utilities**:
- `include/Shared.h` (23 lines) - Public interfaces
- `Shared.m` (36 lines) - App Group NSUserDefaults, darwin notifications, logging

**Settings UI**:
- `BackgroundActionPrefs/BAListController.m` (55 lines) - PSListController subclass
- `BackgroundActionPrefs/Resources/Root.plist` - PreferenceLoader UI definition
- `BackgroundActionPrefs/Makefile` - Bundle build configuration

**Build Configuration**:
- `Makefile` - 3 separate Tweaks + Preferences bundle
- `control` - Package metadata (com.mikiyan1978.backgroundaction v0.1.0)
- `.plist` filters - Process injection targets (RunningBoard/all/SpringBoard)

### Build & Deployment

```bash
# Requires Theos + LLVM + arm64/arm64e cross-compiler
make package

# Generates:
# - BackgroundActionRB_0.1.0_iphoneos-arm64.deb
# - BackgroundActionApp_0.1.0_iphoneos-arm64.deb
# - BackgroundActionSB_0.1.0_iphoneos-arm64.deb
# (install via SSH + dpkg -i)
```

### Critical Pre-Requisite: iOS Version Validation

**This project is a TEMPLATE and UNVERIFIED**. Before building:

1. class-dump RunningBoard binary on target device:
   ```bash
   class-dump /System/Library/PrivateFrameworks/RunningBoard.framework/runningboardd \
     > runningboardd.h
   ```
   
2. Confirm these selectors exist:
   - `RBSProcessHandle -initWithInstance:lifePort:bundleData:reported:`
   - `RBSProcessHandle -bundleID`, `-pid`
   - `RBProcess -terminateWithContext:`, `-processIdentifier`
   - `RBSProcessState -setTaskState:`, `-bundleID`
   - `SBApplication -deactivate`, `-bundleIdentifier`

3. Find correct `kRBSProcessTaskStateRunning` enum value (hardcoded as `2`, likely varies by iOS)

4. Verify `UIApplication._applicationDidEnterBackground` and `UIScene._didEnterBackground` selectors

**If selectors don't match**: Hook will silently fail to attach (Logos just skips %hooks for non-existent selectors). Debug via `BGALog()` writes to `/var/mobile/BackgroundAction_debug.log`.

### Known Risks

1. **Multi-App Interference**
   - Forcing multiple apps into foreground causes CPU/GPU contention
   - Sustained high power state → thermal throttling, battery drain
   - Audio session conflicts (only one app can control audio output)
   - Jetsam memory pressure mechanism gets distorted

2. **OS Stability**
   - Violates iOS resource scheduling assumptions
   - Can cause other foreground apps to be killed prematurely
   - May interfere with system animations/responsiveness

3. **Platform Rejection**
   - Paid repos (Cyder/Silo/Havoc) may reject for modifying standard OS behavior
   - Can be flagged as "system modification" on store audits

**Mitigation** (NOT YET IMPLEMENTED):
- Limit simultaneous enabled apps to 1 or 2
- Auto-timeout after N minutes of whitelist activation
- Add toggle in Settings to rapidly disable if issues occur

### When to Use BackgroundAction

**Good Use Cases**:
- Single upload app during critical operation
- Music playback with custom processing
- Location tracking for navigation/fitness
- Temporary developer testing

**Bad Use Cases**:
- Keeping 5+ apps alive simultaneously
- Permanent 24/7 whitelist without toggling
- Replacing proper backgroundTaskAssertion usage
- Working around app bugs instead of fixing them

---

## Part 4: Reference Skills & Documentation

Three reusable Theos development skills created:

### 1. `runningboard-immortal-process` Skill
- **321 lines** of implementation guide
- RoadRunner reverse-engineering results
- 3-layer hook architecture explanation
- iOS version portability concerns
- Risk mitigation strategies

### 2. `ios-tweak-paid-release` Skill
- **68 lines** - Platform selection guide
- Cyder vs Silo vs Havoc vs BigBoss comparison
- Submission workflow & approval timelines
- Beta distribution vs public release phases
- Prerequisite Tweaks (ios-tweak-license-protection)

### 3. `ios-tweak-license-protection` Skill
- **93 lines** - Anti-piracy implementation patterns
- 5-tier approach: UDID/ECID lock, identification embedding, expiry, webhook logging, server verification
- Cost/benefit analysis per tier
- Recommendation: 4-point set for beta phase
- Integration with paid-release platforms

---

## Part 5: Lessons Learned & Best Practices

### Why Complete Protection is Impossible

Jailbreak provides:
- ✓ Root user (uid 0)
- ✓ Kernel mode access (via Pegasus-like exploits)
- ✓ Code signing bypass
- ✓ System-wide process access

Therefore:
- Any Lua-local validation (code in this Tweak) can be binary-patched out
- Any file-based check can be spoofed
- State persistence can be bypassed by modifying kernel memory

**Cost-benefit shift**: Instead of trying to build an unbreakable vault, focus on:
- Making piracy require specialized skills (UDID/ECID forgery, binary patching)
- Adding telemetry to detect unauthorized usage
- Revoking individual device access when misuse detected
- Protecting legitimate customers via platform-based authentication (Cyder/Silo)

### Layered Defense in Depth

RunningBoard immortalization is strongest when combined:

```
Layer 1 (Kernel/Daemon Level): Process state fixed at RunningBoard
Layer 2 (App Level): UIKit callbacks suppressed in-process
Layer 3 (UI Level): SpringBoard UI commands blocked

Any single layer can be bypassed, but all three together create friction
```

The goal isn't mathematical impossibility—it's "making it not worth the effort."

### iOS Version Portability

Major pain point: **Sel names and enum values change every iOS version**

- iOS 16.2 RBSProcessState.setTaskState enum != iOS 17.x enum
- iOS 15 UIApplication._applicationDidEnterBackground != iOS 16 UIScene._didEnterBackground
- SpringBoard private APIs shift between versions

**Solutions**:
- Runtime selector validation: `respondsToSelector:` before hooking
- Build with multiple iOS SDK versions, auto-detect at runtime
- class-dump target device to confirm selectors before deployment
- Use Theos conditions: `#ifdef iOS15_OR_LATER`

### Tool-Specific Insights

**Theos/Logos**:
- %hook silently skips if selector doesn't exist (important: no crash, just silent failure)
- %orig must be called if you want original behavior (returning early skips it)
- Logos% syntax is preprocessed to MSHookMessageEx at compile time

**Darwin Notifications**:
- Cross-process communication without IPC boilerplate
- Preferred over NSNotification for daemon ↔ userspace
- Uses kernel notify API (notify_post, notify_register_dispatch)

**Associated Objects**:
- objc_setAssociatedObject for runtime-added properties
- Perfect for process state (doesn't require persistent storage)
- Dies with process (can be re-created on reattach)

---

## Part 6: Next Steps & Future Work

### For BackgroundAction Development

1. **Obtain iOS 16.2+ test device** with Dopamine jailbreak
2. class-dump runningboardd + SpringBoard
3. Confirm selector names match hardcoded values
4. Adjust enum values (kRBSProcessTaskStateRunning)
5. Build: `make package`
6. Test: Single app, then multi-app scenarios
7. Add safeguards (timeout, enable-count limit)

### For Tweak Monetization

1. Decide: beta distribution (self-managed) vs. official release (platform-managed)
2. If beta: implement ios-tweak-license-protection patterns
3. If release: choose platform (Cyder/Silo/Havoc) per ios-tweak-paid-release guide
4. Set up developer account + certificate signing
5. Prepare depiction + screenshots + change log

### For Advanced Developers

- **Explore userspace reboot recovery**: How does launchd coordinate with RunningBoard on RB2_USERREBOOT?
- **Investigate XPC communication**: Can BackgroundAction phone home to launchd for smarter reattachment?
- **PAC/pointer authentication**: arm64e binaries use PAC; does RBProcess.terminateWithContext: have PAC-protected return pointers?
- **Audio session routing**: When multiple processes claim audio, how does AudioSession arbitrate? Can we use priority hints?
- **Memory pressure handling**: Custom jetsam score via kern.extra_mem_pages? Does it survive Respring?

---

## Appendix: File Structure Reference

```
dopamine2-roothide/
├── README.md                           (Main project docs)
├── ... (roothide/bootstrap source)

/home/user/BackgroundAction/            (Theos project template)
├── Makefile                            (3 Tweaks + Preferences)
├── control                             (Package metadata)
├── Shared.m / include/Shared.h         (Shared utilities)
├── RunningBoardHook.xm                 (Layer 1: daemon)
├── AppHook.xm                          (Layer 2: app process)
├── SpringBoardHook.xm                  (Layer 3: SpringBoard)
├── *.plist                             (Injection filters)
├── README.md                           (Build/deployment guide)
└── BackgroundActionPrefs/
    ├── Makefile                        (Bundle config)
    ├── BAListController.{h,m}          (Settings UI)
    └── Resources/Root.plist            (UI definition)

/root/.claude/skills/local/
├── runningboard-immortal-process/SKILL.md   (321 lines)
├── ios-tweak-paid-release/SKILL.md          (68 lines)
└── ios-tweak-license-protection/SKILL.md    (93 lines)
```

---

## References

- **roothide/dopamine2-roothide**: https://github.com/roothide/dopamine2-roothide
- **Theos**: https://theos.dev (build system for iOS tweaks)
- **Logos**: Theos preprocessing for runtime method hooking
- **RunningBoard**: Private Framework in iOS 13+, controls process lifecycle
- **TrollStore**: App sideloading framework for non-jailbroken iOS
- **Cydia Substrate**: Runtime method swizzling framework (MSHookMessageEx)

---

**Document Created**: 2025-09-15
**Session**: Claude Code Analysis
**Status**: Research Complete - Ready for Device Validation Phase
